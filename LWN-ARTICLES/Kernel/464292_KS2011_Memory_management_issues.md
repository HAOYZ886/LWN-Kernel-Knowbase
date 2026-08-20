---
title: "KS2011: Memory management issues"
url: https://lwn.net/Articles/464292/
date: "October 24, 2011"
category: "Memory management-Conference sessions"
author: "By Jonathan Corbet October 24, 2011 2011 Kernel Summit coverage"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 24, 2011

* * *

[2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>) coverage 

Mel Gorman started off the memory management session at the 2011 Kernel Summit by noting that a number of good things have happened in the last year. There has been a lot of work on the memory controller for control groups, and the feature is starting to see some serious use. Transparent huge pages went in, along with a lot of fixes and optimizations. The memory management developers have been busy, and they have gotten a lot done. 

[![\[Mel Gorman and
Marek Szyprowski\]](https://static.lwn.net/images/conf/2011/lc-europe/MelAndMarek-sm.jpg)](<https://lwn.net/Articles/464314/>) On the down side, the complexity of the memory management subsystem is getting "severe." A recent problem involving page migration took three core developers to solve. Nobody really knows how the whole thing is implemented, and review bandwidth is a big problem. 

There are, he said, a lot of contentious patches that developers should be paying attention to currently. They include: 

  * The [idle page tracking](<https://lwn.net/Articles/461461/>) patches have been through a number of review cycles. They have not always been received entirely well, but they'll be back again. 

  * Some changes to the slab shrinker API have been around for a while; they are currently suffering from a lack of review. 

  * A cgroup controller putting limits on TCP buffer sizes got through a few rounds of review, only to be slapped down by the networking developers at merge time. They added overhead to the networking fast paths that was not considered acceptable and will need to be reworked. 

  * The [I/O-less writeback throttling patches](<https://lwn.net/Articles/456904/>) were "seriously assaulted" in the review process. But they were reworked in response and now look like an overall success story. People have stopped complaining about them, but they have not yet been merged due to a fear of massive disruptions to certain (unknown) workloads. So, Mel wondered, is it time to just merge the patches and see what happens? A call for objections received none, so these patches may go in as soon as 3.2. 

  * A [set of patches](<https://lwn.net/Articles/443241/>) unifying the LRU list used within and outside of the memory controller remains out there. 

  * [Swapping over NBD and NFS](<https://lwn.net/Articles/439298/>) remains a requested feature; the patches are not popular, but the distributors are shipping them in their kernels anyway. Swap over iSCSI is likely to be added in the near future. There is a clear demand for this feature; it will probably have to go in at some point. 

  * Then, there is the [contiguous memory allocator](<https://lwn.net/Articles/447405/>) (CMA) patch set. After it had been through several reworks, Mel finally got around to reviewing it and "slammed" it. The core idea is good, but he didn't much like the implementation. There are drivers needing this feature and they are not going to go away, so something CMA-like needs to get in one way or another. There was a lot of talk about integrating CMA functionality with other parts of the kernel, including hugetlbfs and the shadow memory map used by the memory controller, but it is not clear how practical those ideas are. 

  * A rework of the DMA mapping API to make better use of I/O memory management units, especially those attached to specific devices. It seems that this job should not be too hard to do. 

Last year's summit included a lot of discussion about writeback, which was seen as the biggest problem at the time. How is it looking now? Mel said that a lot of things have been improved; in particular, the kernel has gotten smarter about how it uses the `congestion_wait()` functionality, which is a big hammer to use when trying to control writeback. There are a lot of new tracepoints for debuggability, and, in 3.2, there will be no more writeback done from direct reclaim - news that was received with applause. The kswapd process still has to initiate some writeback; doing otherwise causes performance problems on NUMA systems. The addition of the I/O-less throttling patches should also help a lot. 

The memory management developers, in other words, have been busy and will continue to be so for a while yet. But they appear to be making some real progress on the problems that have been affecting recent kernels. 

Next: [Preemption disable and verifiable APIs](<https://lwn.net/Articles/464294/>)
