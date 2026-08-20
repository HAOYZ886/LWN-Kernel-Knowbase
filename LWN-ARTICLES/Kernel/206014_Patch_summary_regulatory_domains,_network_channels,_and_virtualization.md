---
title: "Patch summary: regulatory domains, network channels, and virtualization"
url: https://lwn.net/Articles/206014/
date: "October 25, 2006"
category: "KVM; Networking-Channels; Networking-Wireless; Virtualization-KVM"
author: 
---

Here's a quick look at a few patches have been posted recently. 

### 802.11 regulatory domains

Standard wisdom says that putting policy decisions into the kernel is generally a bad idea. Policies implemented in kernel space limit the flexibility of the system, potentially keeping user-space from doing everything it could possibly accomplish. There are times, however, when that is exactly what one might want to do. 

Wireless networking presents a number of challenges for the kernel. One of them is imposed entirely from the outside: anything which can transmit tends to be heavily regulated. So wireless networking adapters must not transmit on unauthorized frequencies or at power levels above those allowed by law. Needless to say, the applicable rules vary from one jurisdiction to the next, making it impossible to work with a single set of constraints, especially if one wants to use the hardware to its full, legal potential in any given country. The need to adhere to regulatory constraints is one of the favorite reasons given by wireless adapter vendors when asked why they cannot release programming information for their hardware. 

Luis Rodriguez is trying to address regulatory issues with [a patch set](<http://lwn.net/Articles/205988/>) implementing regulatory domain information in the kernel (and in the Devicescape 802.11 stack in particular). At this point, the work is just infrastructure which tracks the constraints imposed by any given domain and the current domain under which the system is operating. Actually implementing compliance with the current domain has been left for a future exercise - there are some 802.11 stack issues which need to be resolved first. 

If this patch set is eventually accepted, there will be a single framework by which all wireless adapters can be operated in a legal manner, wherever the computer might happen to be located. Beyond doing the right thing with regard to the spectrum, Luis hopes that this mechanism might be enough to satisfy the various regulatory agencies that Linux has its act together in this regard - and that vendors will no longer feel the need to keep their programming information secret. Luis, it seems, is an optimistic sort of person. 

### Network channels

Meanwhile, things have been quiet for a while on the network channels front. But that does not mean that nothing has been happening. As proof, consider that Evgeniy Polyakov has just surfaced with [a new net channels patch](<http://lwn.net/Articles/205316/>) which, he claims, can scale significantly better than the current networking implementation. 

This version of network channels focuses more on the user-space interface side of the problem, leaving most of the kernel infrastructure work for another time. To that end, it adds a new system call, `netchannel_control()`, to hook up channel functionality to user-space code. `netchannel_control()` is another one of those multiplexer interfaces that Evgeniy seems to favor; it functions like an `ioctl()` call with three core operations: 

  * `NETCHANNEL_CREATE` creates a new channel bound to given local and remote addresses. There is also a "type" specification which describes how the channel operates with user space. 

  * `NETCHANNEL_SEND` will send a packet out on the network. 

  * `NETCHANNEL_RECV` blocks until an incoming packet is received, then passes that packet to user space. 

The kernel side of the implementation, for now, is simple and straightforward: a `NETCHANNEL_SEND` call will allocate an `sk_buff` structure and fill it with user data with `copy_from_user()`; the packet is then sent on its way via the network stack in the usual manner. The design envisions adding other, faster ways of moving data around - using [Evgeniy's network allocator mechanism](<http://lwn.net/Articles/195416/>), for example - in the future. 

The current patch adds [a user-space network stack](<http://tservice.net.ru/~s0mbre/old/?section=projects&item=unetstack>) which uses the new netchannel mechanism. It claims to handle TCP and UDP currently, with a number of the expected features; there is a "socket-like interface" presented to applications. There has been no public reaction to this patch set so far, so it is hard to say whether it makes sense to the other network developers or not. Evgeniy appears to be a persistent sort of person, however, so expect to see this code again. 

### /dev/kvm

Finally, [this large patch set](<http://lwn.net/Articles/205580/>) posted by Avi Kivity may stir things up a bit in the virtualization area. These patches implement support for Intel's virtualization extensions (AMD support is said to be forthcoming), allowing Linux systems to easily run virtual machines without the need for a full hypervisor like Xen. It should be noted that the patch set includes a fair amount of Xen code, though. 

With this patch set added, a Linux system implements a new device called `/dev/kvm`. Opening this device creates a new virtual machine which can then be manipulated with a set of `ioctl()` calls. One important operation creates virtual CPUs for this machine; currently only a single virtual CPU is supported. There is an operation which adds a memory region to the client machine. Memory is organized into "slots" modeled after the physical slots on a motherboard; they are useful for setting up structures like the memory hole at 640K still found on PC-type systems. Other operations allow for the creation of page table entries in the client, manipulating virtual machine registers, intercepting privileged operations, and actually running a program in the client. A set of debugging operations is provided as well. 

There is a fair amount of interest in this patch set; it looks like it could be a (relatively!) simple way of adding hardware virtualization support to the kernel. One comment which has been posted remarks on the similarities between this functionality and the work which has been done to support the "synergistic processing units" (SPUs) on the Cell architecture. The SPU support, which has been in the kernel since 2.6.16, uses a special-purpose filesystem (rather than `ioctl()`) to control the clients. Any sort of merger between these two subsystems would thus likely involve the `/dev/kvm` interface being changed. So this patch set could change quite a bit as it heads toward eventual inclusion.
