---
title: The return of network block device deadlock prevention
url: https://lwn.net/Articles/195416/
date: "August 14, 2006"
category: "Memory management-Out-of-memory handling; Networking"
author: 
---

Just over one year ago, LWN [covered a patch set](<http://lwn.net/Articles/146689/>) aimed at preventing potential deadlocks in the network subsystem. The problem being addressed can come about when the system is using a block (disk) device which is located on the other side of a network link. When the system runs short on memory, one of the things it must do is to write dirty pages back to disk, allowing that memory to be reused for other purposes. But writing to a network disk can require memory allocations in its own right - a need which comes at the worst possible time. This particular problem, which also arises with locally-attached drives, has been solved for a while by keeping a small memory reserve specifically for block I/O operations. 

Network-attached drives have an additional problem, however, in that no write can be considered complete until an acknowledgment has been received from the remote device. Receiving that acknowledgment requires that the system be able to receive (and process) network packets - and that can require unbounded amounts of memory. There may be any amount of incoming network data which has nothing to do with outstanding block I/O requests, and that data can make it impossible to receive the packets which the memory-constrained system is so desperately waiting to receive. The deadlock avoidance patch made some changes aimed at ensuring that the system could always receive and process incoming block I/O traffic. 

A year later, this patch set has [resurfaced](<http://lwn.net/Articles/195308/>). The original author (Daniel Phillips) has stepped aside, and Peter Zijlstra has taken the lead. In many ways, the current version of the patch resembled its predecessors, but there have been enough changes to warrant a new look. 

The patch still works by enlarging the emergency reserve area maintained by the core page allocator. There is a GFP flag (`__GFP_MEMALLOC`) which allows a particular allocation call to be satisfied out of the reserve, if necessary. The core idea is to use this reserve to receive vital incoming network packets without allowing it to be overrun with useless stuff. 

To that end, code which is performing block I/O over a network connection sets the `SOCK_MEMALLOC` flag on its socket(s). Previous versions of the patch would then set a flag on any associated network interfaces to indicate that block I/O was passing through that interface, but the current version skips that step. Instead, any attempt to allocate an `sk_buff` (packet) structure from a network device driver will dip into the memory reserves if need be. Thus, as long as the reserves hold out, the system will always be able to allocate buffers for incoming packets. 

The key is to receive the important packets without exhausting the reserves with useless data (streaming video from LinuxWorld keynotes, say). To that end, the networking code is patched to check for the `SOCK_MEMALLOC` flag as soon as possible after the socket for each incoming packet is identified. If that flag is not set, and the incoming packet is using memory from the reserves, the packet will be dropped immediately, freeing its memory for other uses. So packets related to block I/O are received and processed as usual; just about everything else gets dropped at the earliest possible moment. 

The latest version of the patch includes a new memory allocator, called SROG, which is used for handling reserve memory. It is intended to be fast and simple, and to release memory back to the system as quickly as possible. To that end, it tries to group related allocations together, and it isolates each group of allocations (generally the `sk_buff` structure and its associated data area) onto their own pages. So every time a packet is released, its associated memory immediately becomes available to the system as a whole. 

This patch set is proving to be a bit of a hard sell, however. The deadlock scenario is seen as being relatively unlikely - there have not been streams of bug reports on this topic - and, in most cases, it can be avoided simply by swapping to a local disk. The set of systems whose owners can afford fancy network storage arrays, but where those same owners are unable to invest in a local disk for swapping, is thought to be small. Making the networking layer more complex to address this particular problem does not appeal to everybody. 

Networking maintainer David Miller [would like to see](<https://lwn.net/Articles/195430/>) a different sort of approach to network memory allocations: 

I think there is more profitability from a solution that really does something about "network memory", and doesn't try to say "these devices are special" or "these sockets are special". Special cases generally suck. 

We already limit and control TCP socket memory globally in the system. If we do this for all socket and anonymous network buffer allocations, which is sort of implicity in Evgeniy's network tree allocator design, we can solve this problem in a more reasonable way. 

This comment refers to Evgeniy Polyakov's [network memory allocator patch](<http://lwn.net/Articles/195292/>), recently posted for consideration. This work is in a highly transitional state and is a little hard to read. The core, however, is this: it is (yet another) separate memory allocator, oriented toward the needs of the networking system. It is designed to keep memory allocations local to a single CPU, so each processor has its own set of pages to hand out. Allocated objects are packed as tightly as possible, minimizing internal fragmentation. There is no recourse to the system memory allocator in the current design, so, when a particular processor runs out, allocations will fail. Memory exhaustion in the rest of the system will not affect the network allocator, however. The author claims improved networking performance: 

Benchmarks with trivial epoll based web server showed noticeable (more than 40%) improvements of the request rates (1600-1800 requests per second vs. more than 2300 ones). It can be described by more cache-friendly freeing algorithm, by tighter objects packing and thus reduced cache line ping-pongs, reduced lookups into higher-layer caches and so on. 

This code is also written with an eye toward mapping networking buffers directly into user space, perhaps in conjunction with a future network channel implementation. 

The network allocator patch clearly has the eye of the networking maintainer at the moment. That code is fairly far from being ready to merge, however, and not everybody agrees that it solves all of problems. So this is a discussion which could go on for some time yet.
