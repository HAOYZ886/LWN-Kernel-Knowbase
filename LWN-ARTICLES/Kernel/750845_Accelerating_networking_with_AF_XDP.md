---
title: Accelerating networking with AF_XDP
url: https://lwn.net/Articles/750845/
date: "April 9, 2018"
category: "Networking-eXpress Data Path XDP"
author: "By Jonathan Corbet April 9, 2018"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 9, 2018

The Linux network stack does not lack for features; it also performs well enough for most uses. At the highest network speeds, though, any overhead at all is too much; that has driven the most demanding users toward specialized, user-space networking implementations that can outperform the kernel for highly constrained tasks. The express data path (XDP) development effort is an attempt to win those users back, with some apparent success so far. With the posting of the [`AF_XDP` patch set](<https://lwn.net/Articles/750293/>) by Björn Töpel, another piece of the XDP puzzle is coming into focus. 

The core idea behind the XDP initiative is to get the network stack out of the way as much as possible. While the network stack is highly flexible, XDP is built around a bare-bones packet transport that is as fast as it can be. When a decision needs to be made or a packet must be modified, XDP will provide a hook for a user-supplied BPF program to do the work. The result combines minimal overhead with a great deal of flexibility, at the cost of a little "some assembly required" label on the relevant man pages. For users who count every nanosecond of packet-processing overhead (to the point that the 4.17 kernel will include some painstaking enhancements to the BPF JIT compiler that reduces the size of the generated code by 5%), figuring out how to put the pieces together is worth the trouble. 

The [earliest XDP work](<https://lwn.net/Articles/682538/>) enabled the loading of a BPF program into the network interface device driver, with the initial use case being a program that dropped packets as quickly as possible. That may not be the most exciting application, but it is a useful feature for a site that is concerned about fending off distributed denial-of-service attacks. Since then, XDP has gained the ability to perform simple routing (retransmitting a packet out the same interface it arrived on) and, for some hardware, to offload the BPF program into the interface itself. 

There are limits, though, to what can be done in the context of a network-interface driver; for such cases, `AF_XDP` is intended to connect the XDP path through to user space. It can be thought of as being similar to the [`AF_PACKET`](<http://man7.org/linux/man-pages/man7/packet.7.html>) address family, in that it transports packets to and from an application with a minimum of processing, but this interface is clearly intended for applications that prioritize packet-processing performance above convenience. So, once again, some assembly is required in order to actually use it. 

That assembly starts by calling [`socket()`](<http://man7.org/linux/man-pages/man2/socket.2.html>) in the usual way with the `AF_XDP` address family; that yields a socket file descriptor that can (eventually) be used to move packets. First, however, it is necessary to create an array in user-space memory called a "UMEM". It is a chunk of contiguous memory, divided into equal-sized "frames" (the actual size is specified by the caller), each of which can hold a single packet. By itself, the UMEM looks rather boring: 

> ![\[The UMEM array\]](https://static.lwn.net/images/2018/af_xdp1.png)

After the memory has been allocated by the application, this array is registered with the socket using the `XDP_UMEM_REG` command of the [`setsockopt()`](<http://man7.org/linux/man-pages/man2/getsockopt.2.html>) system call. 

Each frame in the array has an integer index called a "descriptor". To use those descriptors, the application creates a circular buffer called the "fill queue", using the `XDP_UMEM_FILL_QUEUE` `setsockopt()` call. This queue can then be mapped into user-space memory using `mmap()`. The application can request that the kernel place an incoming packet into a specific frame in the UMEM array by adding that frame's descriptor to the fill queue: 

> ![\[The fill queue\]](https://static.lwn.net/images/2018/af_xdp2.png)

Once a descriptor goes into the fill queue, the kernel owns it (and the associated UMEM frame). Getting that descriptor back (with a new packet in the associated frame) requires creating yet another queue (the "receive queue"), with the `XDP_RX_QUEUE` `setsockopt()` operation. It, too, is a circular buffer that must be mapped into user space; once a frame has been filled with a packet, its descriptor will be moved to the receive queue. A call to `poll()` can be used to wait for packets to arrive in the receive queue. 

> ![\[The receive queue\]](https://static.lwn.net/images/2018/af_xdp3.png)

A similar story exists on the transmit side. The application creates a transmit queue with `XDP_TX_QUEUE` and maps it; a packet is transmitted by placing its descriptor into that queue. A call to `sendmsg()` informs the kernel that one or more descriptors are ready for transmission. The completion queue (created with `XDP_UMEM_COMPLETION_QUEUE`) receives descriptors from the kernel after the packets they contain have been transmitted. The full picture looks something like this: 

> ![\[The transmit queue\]](https://static.lwn.net/images/2018/af_xdp4.png)

This whole data structure is designed to enable zero-copy movement of packet data between user space and the kernel, though the current patches do not yet implement that. It also allows received packets to be retransmitted without copying them, since any descriptor can be used for either transmission or reception. 

The UMEM array can be shared between multiple processes. If a process wants to create an `AF_XDP` socket attached to an existing UMEM, it simply passes its socket file descriptor and the file descriptor associated with the socket owning the UMEM to [`bind()`](<http://man7.org/linux/man-pages/man2/bind.2.html>); the second file descriptor is passed in the `sockaddr_xdp` structure. There is only one fill queue and one completion queue associated with the UMEM regardless of how many processes are using it, but each process must maintain its own transmit and receive queues. In other words, in a multi-process configuration, it is expected that one process (or thread) will be dedicated to the management of the UMEM frames, while each of the others takes on one aspect of the packet-handling task. 

There is one other little twist here, relating to how the kernel chooses a receive queue for any given incoming packet. There are two pieces to that puzzle, the first of which is yet another new BPF map type called `BPF_MAP_TYPE_XSKMAP`. This map is a simple array, each entry of which can contain a file descriptor corresponding to an `AF_XDP` socket. A process that is attached to the UMEM can call `bpf()` to store its file descriptor in the map; what is actually stored is an internal kernel pointer, of course, but applications won't see that. The other piece is a BPF program loaded into the driver whose job is to classify incoming packets and direct them to one of the entries in the map; that will cause the packets to show up in the receive queue corresponding to the `AF_XDP` socket in the chosen map entry. 

Without the map and BPF program, an `AF_XDP` socket is unable to receive packets. You _were_ warned that some assembly was required. 

The final piece is a `bind()` call to attach the socket to a specific network interface and, probably, a specific hardware queue within that interface. The interface itself can then be configured to direct packets to that queue if they should be handled by the program behind the `AF_XDP` socket. 

The intended final result is a structure that can enable user-space code to perform highly efficient packet management, with as much hardware support as possible and with a minimum of overhead in the kernel. There are some other pieces that are needed to get there, though. The zero-copy code is clearly one of them; copying packet data between the kernel and user space is fatal in any sort of high-performance scenario. Another one is the [XDP redirect patch set](<https://lwn.net/Articles/750571/>) being developed by Jesper Dangaard Brouer; that functionality is what will allow an XDP program to direct packets toward specific `AF_XDP` sockets. Driver support is also required; that is headed toward mainline for a couple of Intel network interfaces now. 

If it all works as planned, it should become possible to process packets at a much higher rate than has been possible with the mainline network stack so far. This functionality is not something that many developers will feel driven to use, but it is intended to be appealing to those who have resorted to user-space stacks in the past. It is a sign of an interesting direction that kernel development has taken: new functionality is highly flexible, but using it requires programming for the BPF virtual machine.
