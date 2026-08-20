---
title: The multiqueue block layer
url: https://lwn.net/Articles/552904/
date: "June 5, 2013"
category: Block layer
author: "By Jonathan Corbet June 5, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
June 5, 2013

The kernel's block layer is charged with managing I/O to the system's block ("disk drive") devices. It was designed in an era when a high-performance drive could handle hundreds of I/O operations per second (IOPs); the fact that it tends to fall down with modern devices, capable of handling possibly millions of IOPs, is thus not entirely surprising. It has been known for years that significant changes would need to be made to enable Linux to perform well on fast solid-state devices. The shape of those changes is becoming clearer as the multiqueue block layer patch set, primarily the work of Jens Axboe and Shaohua Li, gets closer to being ready for mainline merging. 

The basic structure of the block layer has not changed a whole lot since it was described for 2.6.10 in [_Linux Device Drivers_](<https://lwn.net/Kernel/LDD3/>). It offers two ways for a block driver to hook into the system, one of which is the "request" interface. When run in this mode, the block layer maintains a simple request queue; new I/O requests are submitted to the tail of the queue and the driver receives requests from the head. While requests sit in the queue, the block layer can operate on them in a number of ways: they can be reordered to minimize seek operations, adjacent requests can be coalesced into larger operations, and policies for fairness and bandwidth limits can be applied, for example. 

This request queue turns out to be one of the biggest bottlenecks in the entire system. It is protected by a single lock which, on a large system, will bounce frequently between the processors. It is a linked list, a notably cache-unfriendly data structure especially when modifications must be made — as they frequently are in the block layer. As a result, anybody who is trying to develop a driver for high-performance storage devices wants to do away with this request queue and replace it with something better. 

The second block driver mode — the "make request" interface — allows a driver to do exactly that. It hooks the driver into a much higher part of the stack, shorting out the request queue and handing I/O requests directly to the driver. This interface was not originally intended for high-performance drivers; instead, it is there for stacked drivers (the MD RAID implementation, for example) that need to process requests before passing them on to the real, underlying device. Using it in other situations incurs a substantial cost: all of the other queue processing done by the block layer is lost and must be reimplemented in the driver. 

The multiqueue block layer work tries to fix this problem by adding a third mode for drivers to use. In this mode, the request queue is split into a number of separate queues: 

  * Submission queues are set up on a per-CPU or per-node basis. Each CPU submits I/O operations into its own queue, with no interaction with the other CPUs. Contention for the submission queue lock is thus eliminated (when per-CPU queues are used) or greatly reduced (for per-node queues). 

  * One or more hardware dispatch queues simply buffer I/O requests for the driver. 

While requests are in the submission queue, they can be operated on by the block layer in the usual manner. Reordering of requests for locality offers little or no benefit on solid-state devices; indeed, spreading requests out across the device might help with the parallel processing of requests. So reordering will not be done, but coalescing requests will reduce the total number of I/O operations, improving performance somewhat. Since the submission queues are per-CPU, there is no way to coalesce requests submitted to different queues. With no empirical evidence whatsoever, your editor would guess that adjacent requests are most likely to come from the same process and, thus, will automatically find their way into the same submission queue, so the lack of cross-CPU coalescing is probably not a big problem. 

The block layer will move requests from the submission queues into the hardware queues up to the maximum number specified by the driver. Most current devices will have a single hardware queue, but high-end devices already support multiple queues to increase parallelism. On such a device, the entire submission and completion path should be able to run on the same CPU as the process generating the I/O, maximizing cache locality (and, thus, performance). If desired, fairness or bandwidth-cap policies can be applied as requests move to the hardware queues, but there will be an associated performance cost. Given the speed of high-end devices, it may not be worthwhile to try to ensure fairness between users; everybody should be able to get all the I/O bandwidth they can use. 

This structure makes the writing of a high-performance block driver (relatively) simple. The driver provides a `queue_rq()` function to handle incoming requests and calls back to the block layer when requests complete. Those wanting to look at an example of how such a driver would work can see [null_blk.c](<https://lwn.net/Articles/552911/>) in the `new-queue` branch of Jens's block repository: 

```
git://git.kernel.dk/linux-block.git
```

In the current patch set, the multiqueue mode is offered in addition to the existing two modes, so current drivers will continue to work without change. According to [this paper on the multiqueue block layer design [PDF]](<http://kernel.dk/systor13-final18.pdf>), the hope is that drivers will migrate over to the multiqueue API, allowing the eventual removal of the request-based mode. 

This patch set has been significantly reworked in the last month or so; it has gone from a relatively messy series into something rather cleaner. Merging into the mainline would thus appear to be on the agenda for the near future. Since use of this API is optional, existing drivers should continue to work and this merge could conceivably happen as early as 3.11. But, given that the patch set has not yet been publicly posted to any mailing list and does not appear in linux-next, 3.12 seems like a more likely target. Either way, Linux seems likely to have a much better block layer by the end of the year or so.
