---
title: "Van Jacobson's network channels"
url: https://lwn.net/Articles/169961/
date: "January 31, 2006"
category: "Networking-Channels"
author: 
---

Your editor had the good fortune to see Van Jacobson speak at the 1989 USENIX conference. His talk covered some of the bleeding-edge topics of the time, including TCP slow start algorithms and congestion avoidance. It was the "how Van saved the net" talk (though he certainly did not put it in those terms), and, many years later, the impression from that talk remains. Van Jacobson is a smart guy. 

Unfortunately, attending Van's talk at linux.conf.au this year was not in the program. Fortunately, [David Miller was there](<http://vger.kernel.org/~davem/cgi-bin/blog.cgi/2006/01/27#vj_channels>) and listening carefully. Van has figured out how the next round of networking performance improvements will happen, and he has the numbers to prove it. Expect some very interesting (and fundamental) changes in the Linux networking stack as Van's ideas are incorporated. This article attempts to cover the fundamentals of Van's scheme (called "channels") based on David's weblog entry and [Van's slides [PDF]](<http://www.lemis.com/grog/Documentation/vj/lca06vj.pdf>). 

Van, like many others, points out that the biggest impediment to scalability on contemporary hardware is memory performance. Current processors can often execute multiple instructions per nanosecond, but loading a cache line from memory still takes 50ns or more. So cache behavior will often be the dominant factor in the performance of kernel code. That is why simply making code smaller often makes it faster. The kernel developers understand cache behavior well, and much work has gone into improving cache utilization in the kernel. 

The Linux networking stack (like all others) does a number of things which reduce cache performance, however. These include: 

  * Passing network packets through multiple layers of the kernel. When a packet arrives, the network card's interrupt handler begins the task of feeding the packet to the kernel. The remainder of the work may well be performed at software interrupt level within the driver (in a tasklet, perhaps). The core network processing happens in another software interrupt. Copying the data (an expensive operation in itself) to the application happens in kernel context. Finally the application itself does something interesting with the data. The context changes are expensive, and if any of these changes causes the work to move from one CPU to another, a big cache penalty results. Much work has been done to improve CPU locality in the networking subsystem, but much remains to be done. 

  * Locking is expensive. Taking a lock requires a cross-system atomic operation and moves a cache line between processors. Locking costs have led to the development of lock-free techniques like [seqlocks](<http://lwn.net/Articles/22818/>) and [read-copy-update](<http://lwn.net/Articles/37889/>), but the the networking stack (like the rest of the kernel) remains full of locks. 

  * The networking code makes extensive use of queues implemented with doubly-linked lists. These lists have poor cache behavior since they require each user to make changes (and thus move cache lines) in multiple places. 

To demonstrate what can happen, Van ran some [netperf](<http://www.netperf.org/netperf/NetperfPage.html>) tests on an instrumented kernel. On a single CPU system, processor utilization was 50%, of which 16% was in the socket code, 5% in the scheduler, and 1% in the application. On a two-processor system, utilization went to 77%, including 24% in the socket code and 12% in the scheduler. That is a worst case scenario in at least one way: the application and the interrupt handler were configured to run on different CPUs. Things will not always be that bad in the real world, but, as the number of processors increases, the chances of the interrupt handler running on the same processor as any given application decrease. 

The key to better networking scalability, says Van, is to get rid of locking and shared data as much as possible, and to make sure that as much processing work as possible is done on the CPU where the application is running. It is, he says, simply the end-to-end principle in action yet again. This principle, which says that all of the intelligence in the network belongs at the ends of the connections, doesn't stop at the kernel. It should continue, pushing as much work as possible out of the core kernel and toward the actual applications. 

The tool used to make this shift happen is the "net channel," intended to be a replacement for the socket buffers and queues used in the kernel now. Some details of how channels are implemented can be found in Van's slides, but all that really matters is the core concept: a channel is a carefully designed circular buffer. Properly done, circular buffers require no locks and share no writable cache lines between the producer and the consumer. So adding data to (or removing data from) a net channel will be a fast, cache-friendly operation. 

As a first step, channels can be pushed into the driver interface. A network driver need no longer be aware of `sk_buff` structures and such; instead, it simply drops incoming packets into a channel as they are received. Making this change cuts the CPU utilization in the two-processor case back to 58%. But things need not stop there. A next logical step would be to get rid of the networking stack processing at softirq level and to feed packets directly into the socket code via a channel. Doing that requires creating a separate channel for each socket and adding a simple packet classifier so that the driver knows which channel should get each packet. The socket code must also be rewritten to do the protocol processing (using the existing kernel code). That change drops the overall CPU utilization to 28%, with the portion spent at softirq level dropping to zero. 

But why stop there? If one wants to be serious about this end-to-end thing, one could connect the channel directly to the application. Said application gets the packet buffers mapped directly into its address space and performs protocol processing by way of a user-space library. This would be a huge change in how Linux does networking, but Van's results speak for themselves. Here is his table showing the percentage CPU utilization for each of the cases described above: 

> | Total CPU| Interrupt| SoftIRQ | Socket| Locks| Sched| App.  
> ---|---|---|---|---|---|---|---  
> 1 CPU | 50 | 7 | 11 | 16 | 8 | 5 | 1  
> 2 CPUs | 77 | 9 | 13 | 24 | 14 | 12 | 1  
> Driver channel | 58 | 6 | 12 | 16 | 9 | 9 | 1  
> Socket channel | 28 | 6 | 0 | 16 | 1 | 3 | 1  
> App. channel | 14 | 6 | 0 | 0 | 0 | 2 | 5  
  
The bottom line (literally) is this: processing time for the packet stream dropped to just over 25% of the previous single-CPU case, and less than 20% of the previous two-CPU behavior. Three layers of kernel code have been shorted out altogether, with the remaining work performed in the driver interrupt handler and the application itself. The test system running with the full application channel code was able to handle twice the network bandwidth as an unmodified system - with the processors idle most of the time. 

Linux networking hackers have always been highly attentive to performance issues, so numbers like these are bound to get their attention. Beyond performance, however, this approach promises simpler drivers and a reasonably straightforward transition between the current stack and a future stack built around channels. A channel-based user-space interface will make it easy to create applications which can send and receive packets using any protocol. If Van's results hold together in a "real-world" implementation, the only remaining question would be: when will it be merged so the rest of us can use it?
