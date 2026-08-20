---
title: "Toward more robust network-based block I/O"
url: https://lwn.net/Articles/146689/
date: "August 8, 2005"
category: "Memory management-Out-of-memory handling; Networking"
author: 
---

One thing which came out of [this year's Kernel Summit](<https://lwn.net/Articles/KernelSummit2005/>) is that the kernel still does not deal well with network-based block devices when memory gets tight. If the system is full of dirty memory, the kernel must write some of those dirty pages to their backing store so that the memory may be reused. But the act of writing that data over the network can require the allocation of more memory. Even worse, completing network-based I/O requires the ability to receive the acknowledgment packets back from the remote device. Not only does that packet reception require memory, but the system must contend with the fact that the network could also be the source of vast numbers of packets which are completely unrelated to the problem at hand. If the system cannot find a way to receive the packets it needs while ignoring unrelated packets, extreme memory pressure will eventually lead to a lockup. 

Solving this problem is hard. At the Summit, Linus suggested that it might not even make sense to try; instead, users should be directed toward I/O hardware which does not present this sort of problem. In reality, however, Linux will do its best to support network-based block devices. Daniel Phillips has recently been working on [a patch](<http://lwn.net/Articles/146652/>) which tries to make some progress in that direction. 

Like many before him, Daniel bases his approach on the use of preallocated memory pools - a chunk of memory which is set aside for use when no other memory is available. Daniel has tried to take things a little further by quantifying how much memory should be set aside. To that end, each network driver should, when an interface is brought up, make a call to: 

```
int adjust_memalloc_reserve(int pages);
```

Where `pages` is the number of pages required to be able to continue to receive packets on the given interface. A helper function, `estimate_skb_pages()`, can come up with a guess for how many pages will be required to hold a given number of packets with a specified maximum size. The call to `adjust_memalloc_reserve()` will cause the virtual memory subsystem to set aside the given number of pages for emergency use by the driver. In this way, it is hoped, the system will reserve a sufficient amount of memory without being overly wasteful. 

Memory can be allocated from the reserve by adding the new `__GFP_MEMALLOC` flag to the allocation request. A new networking helper function, `dev_memalloc_skb()`, will use that flag if necessary to obtain a packet. Before doing so, however, it checks a count of packets allocated from the reserve; no interface is allowed to allocate beyond a maximum count, which defaults to 50\. Unlike previous versions of the patch, the current code does not attempt to track which packets, in particular, were allocated from reserve memory. Any packets which originate from a given device will, when returned to the system, be credited to that device's reserve. 

A longstanding problem with the reserve approach is that, if one is not careful, the reserve simply gets depleted and the system runs out of memory anyway. In a situation where memory use is not entirely within the system's control - when dealing with incoming network data, for example - this sort of depletion is especially likely. Your system may be doing its best to flush dirty pages to your home iSCSI array, but the network memory reserves are full of incoming music being downloaded by your children, so the entire system comes to a halt. Such an outcome may please the RIAA, but the kernel developers are trying to satisfy a different audience. 

Daniel's answer to this problem is to add a special flag to network sockets which are involved in block I/O. Only sockets marked with `SOCK_MEMALLOC` are entitled to use packet memory from the reserves. When the packet arrives on the interface, the system cannot know whether it is useful or not, so that packet must be received (possibly using reserve memory) and fed into the system in the usual way. The protocol code, however, is expected to check each packet to see whether it comes from a device which is currently using reserve memory. If so, and the packet does not belong to a suitably-marked socket, that packet is to be dropped immediately. In this way, it is hoped, the system will be able to focus its remaining resources on recovering from its memory crunch. 

This approach may have some promise. This patch needs some work, however, before it is ready for serious stress testing. Once it has been worked into shape, the patch can be applied to a suitably-equipped system, which can then be pushed into a state of serious memory pressure. That point has been the downfall of a number of other approaches to this problem; whether Daniel's work is up to this test remains to be seen.
