---
title: Replacing congestion_wait()
url: https://lwn.net/Articles/873672/
date: "October 25, 2021"
category: Memory management
author: "By Jonathan Corbet October 25, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 25, 2021

Memory management is a balancing act in a number of ways. The kernel must balance the needs of current users of memory with anticipated future needs, for example. The kernel must also balance the act of reclaiming memory for other uses, which can involve writing data to permanent storage, with the rate of data that the underlying storage devices are able to accept. For years, the memory-management subsystem has used storage-device congestion as a signal that it should slow down reclaim. Unfortunately, that mechanism, which was a bit questionable from the beginning, has not worked in a long time. Mel Gorman is now trying to fix this problem with [a patch set](<https://lwn.net/ml/linux-kernel/20211022144651.19914-1-mgorman@techsingularity.net/>) that moves the kernel away from the idea of waiting on congestion. 

#### The congestion_wait() surprise

When memory gets tight, the memory-management subsystem must reclaim in-use pages for other uses; that, in turn, requires writing out the contents of any pages that have been modified. If the block device to which the pages are to be written is overwhelmed with traffic, though, there is little value to making the pile of I/O requests even deeper. Back in the dark and distant pre-Git past (2002), Andrew Morton [proposed](<https://lwn.net/Articles/9519/>) the addition of a congestion-tracking mechanism for block devices; if a given device was congested, the memory-management subsystem would hold off on creating new I/O requests (and throttle — slow down — processes needing more memory) until the congestion eased. This mechanism found its way into the [2.5.39](<https://lwn.net/Articles/11130/>) development kernel release in September 2002. 

Over the years since then, the congestion-wait mechanism has moved around and evolved in various ways. The upcoming 5.15 kernel will still include a function called [`congestion_wait()`](<https://elixir.bootlin.com/linux/v5.15-rc6/source/mm/backing-dev.c#L1017>) that blocks the current task until either the congested device becomes uncongested (as signaled by a call to [`clear_bdi_congested()`](<https://elixir.bootlin.com/linux/v5.15-rc6/source/mm/backing-dev.c#L994>)) or a timeout expires. Or, at least, that is the intent. 

As it happens, the main caller of `clear_bdi_congested()` was a function called `blk_clear_congested()`, and that function was [removed](<https://git.kernel.org/linus/a1ce35fa4985>) for the 5.0 kernel release in 2018. With the exception of a few filesystems (Ceph, FUSE, and NFS), nothing has been calling `clear_bdi_congested()` since then, meaning that calls to `congestion_wait()` almost always just sit until the timeout expires, which was _not_ the intent. 

It took another year (until September 2019) for the memory-management developers to [figure this out](<https://lore.kernel.org/linux-mm/20190919034949.GF9880@bombadil.infradead.org/>), at which point block subsystem maintainer Jens Axboe [let it be known](<https://lore.kernel.org/linux-mm/45d8b7a6-8548-65f5-cccf-9f451d4ae3d4@kernel.dk/>) that: 

> Congestion isn't there anymore. It was always broken as a concept imho, since it was inherently racy. We used the old batching mechanism in the legacy stack to signal it, and it only worked for some devices. 

The race-prone nature of the congestion infrastructure was actually noted by Morton in his original proposal; a task could check a device and see that it is not congested, but the situation could change before that task gets around to queuing its I/O requests. Congestion tracking also gets harder to do accurately as the length of the command queues supported by storage devices increases. So the block developers decided to get rid of the concept in 2018. Unfortunately, nobody there told the memory-management developers, a fact that led to [a grumpy comment](<https://lore.kernel.org/linux-mm/20190924121643.GO23050@dhcp22.suse.cz/>) from Michal Hocko when the situation came to light. 

This is an unfortunate case of one hand not knowing what the other is doing; it has resulted in reduced memory-management performance for years. But kernel developers tend not to sit around and recriminate over such things; instead they started thinking about how to solve this problem. They must have thought fairly hard, since that process took another two years before patches started coming to light. 

#### Moving beyond congestion

Gorman's patch set starts by noting that ""even if congestion throttling worked, it was never a great idea"". There are a number of things that can slow down the reclaim process. One of those — too many pages under writeback overwhelming the underlying device — might be addressed by a (properly working) congestion-wait mechanism, but other problems would not be. So the patch set takes out all of the `congestion_wait()` calls and replaces them with a different set of heuristics: 

  * There are places in the memory-management subsystem where reclaim will be throttled. For example, if the kswapd thread finds pages currently being written back that have been marked for immediate reclaim, it indicates that those pages have cycled all the way through the least-recently-used (LRU) lists before they can be written to the backing store. When that happens, tasks performing reclaim will be throttled for some time. Rather than waiting for the non-existent "congestion is gone" signal, though, reclaim will stall until enough pages on the current NUMA node have been written to indicate that progress is being made. 

Note that some threads — kernel threads and I/O workers in particular — will not be throttled in this case; their work may be needed to clear the backlog. 
  * Many memory-management operations, such as compaction and page migration, require "isolating" the pages to be operated on. Isolation, in this case, refers to removing the page from any LRU lists. The reclaim process, too, involves isolating pages before they can be written. If many tasks end up performing direct reclaim, they can isolate a lot of pages that may take some time to fully reclaim; if the kernel is isolating pages more quickly than they can be reclaimed, the effort is, in the end, wasted. 

The kernel already throttles reclaim if the number of isolated pages becomes too large, but that throttling waits (or tries to wait) on congestion. Gorman [noted](<https://lwn.net/ml/linux-kernel/20211022144651.19914-3-mgorman@techsingularity.net/>): ""This makes no sense, excessive parallelisation has nothing to do with writeback or congestion"". The new code instead contains a wait queue for tasks that have been throttled while performing reclaim as the result of too many isolated pages; they will be awakened when the number of isolated pages drops or a timeout occurs. 
  * Sometimes, a thread performing reclaim may find that it is making little progress; it scans a lot of pages, but succeeds in reclaiming few of them. This can happen as the result of too many references to the pages it is working on or various other factors. With this patch set, threads that are making insufficient progress in reclaim will be throttled until some progress is made somewhere in the system. Specifically, the kernel will wait until running reclaim threads are successful with at least 12% of the pages they scan before waking threads that were not making progress. This should reduce the amount of time wasted in unproductive reclaim efforts. 
  * Writeback efforts will also be throttled if an attempt to write out dirty pages fails due to a lack of memory. The throttling, in this case, lasts until a number of pages have been successfully written back (or a timeout occurs, as usual). 

Most of the timeout durations are set to one-tenth of a second. The wait for the number of isolated pages to drop, though, is one-fiftieth of a second on the reasoning that this situation should change quickly. [The patch setting these timeouts](<https://lwn.net/ml/linux-kernel/20211022144651.19914-7-mgorman@techsingularity.net/>) notes that they are ""pulled out of thin air"", but they are something to start with until somebody finds a way to come up with better values. As a first step in that direction, the no-progress timeout was later [changed](<https://lwn.net/ml/linux-kernel/20211022144651.19914-8-mgorman@techsingularity.net/>) to a half-second after benchmark results showed that it was expiring too quickly. 

The patch set is accompanied by an extensive set of benchmark results; as part of the testing, Gorman added a new "stutterp" test designed to exhibit reclaim problems. The results vary quite a bit but are generally positive; one test shows an 89% reduction in system CPU time, for example. Gorman concluded: 

> Bottom line, the throttling appears to work and the wakeup events may limit worst case stalls. There might be some grounds for adjusting timeouts but it's likely futile as the worst-case scenarios depend on the workload, memory size and the speed of the storage. A better approach to improve the series further would be to prioritise tasks based on their rate of allocation with the caveat that it may be very expensive to track. 

These patches have been through five revisions to date with various changes happening along the way. It is hard to imagine a scenario where this work does not eventually get merged into the mainline; the current code is demonstrably broken, after all. But this kind of core memory-management change is hard to merge; the variety of workloads is so great that there is certainly something out there that will regress when heuristics are changed in this way. So, while something like this seems likely to be accepted, one never knows how many timeouts will expire before that happens.
