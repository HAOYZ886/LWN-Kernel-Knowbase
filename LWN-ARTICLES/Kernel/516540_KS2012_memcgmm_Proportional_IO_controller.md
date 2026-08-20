---
title: "KS2012: memcg/mm: Proportional I/O controller"
url: https://lwn.net/Articles/516540/
date: "September 17, 2012"
category: "Block layer-IO scheduling"
author: "By Michael Kerrisk September 17, 2012 2012 Kernel Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Michael Kerrisk**  
September 17, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

During the 2012 Kernel Summit [memcg/mm minisummit](<https://lwn.net/Articles/516439/>), Fengguang Wu initiated a discussion on improving the implementation of the proportional I/O controller. This controller allows the user to assign I/O weights for each cgroup (see the kernel source file [`Documentation/cgroups/blkio-controller.txt`](<https://lwn.net/Articles/516428/>) for some background). The controller works well for direct I/O, since the CFQ (Completely Fair Queuing) I/O scheduler has one sync queue for each blkio cgroup. However, it comes up short when the blkio cgroups also submit buffered writes, because the buffered write I/Os are currently all mixed into one single global CFQ queue. 

The straightforward solution, proposed by Tejun Heo, is to split up the global CFQ queue by cgroup, so that the CFQ scheduler can easily schedule the per-cgroup sync/async queues according to the per-cgroup I/O weights. Unfortunately, the split will lead to smaller I/O sizes and an increase in the number of `PG_writeback` pages present in the async CFQ queue(s) at any one time, hurting I/O performance, system responsiveness, and huge-page allocations. 

Fengguang was not happy with Tejun's proposed solution. Instead, he proposes a "zero-cost" and scalable scheme: 

  1. Keep the one async CFQ queue. 

  2. Support per-cgroup buffered write weights in `balance_dirty_pages()` (BDP). 

  3. Run a user-space daemon that updates the CFQ/BDP weights every second, so that the resulting I/O throughput meets the user-desired per-cgroup I/O weights in the long term. Fengguang noted, however, that this part may be challenging to get right. 

There were relatively few conclusive comments on both proposals, presumably because of the inherent complexities of the problem and the fact that some active figures in this area were not present for the meeting. 

[Next: Shared-memory accounting in memory cgroups](<https://lwn.net/Articles/516541/>)
