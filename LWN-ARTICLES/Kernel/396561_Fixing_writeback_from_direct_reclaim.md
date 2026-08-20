---
title: Fixing writeback from direct reclaim
url: https://lwn.net/Articles/396561/
date: "July 20, 2010"
category: "Memory management-Writeback"
author: "By Jonathan Corbet July 20, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 20, 2010

"Writeback" is the process of writing the contents of dirty memory pages back to their backing store, where that backing store is normally a file or swap area. Proper handling of writeback is crucial for both system performance and data integrity. If writeback falls too far behind the dirtying of pages, it could leave the system with severe memory pressure problems. Having lots of dirty data in memory also increases the amount of data which may be lost in the event of a system crash. Overly enthusiastic writeback, on the other hand, can lead to excessive I/O bandwidth usage, and poorly-planned writeback can greatly reduce I/O performance with excessive disk seeks. Like many memory-management tasks, getting writeback right is a tricky exercise involving compromises and heuristics. 

Back in April, LWN [looked at a specific writeback problem](<http://lwn.net/Articles/384093/>): quite a bit of writeback activity was happening in direct reclaim. Normally, memory pages are reclaimed (made available for new uses, with data written back, if necessary) in the background; when all goes well, there will always be a list of free pages available when memory is needed. If, however, a memory allocation request cannot be satisfied from the free list, the kernel may try to reclaim pages directly in the context of the process performing the allocation. Diverting an allocation request into this kind of cleanup activity is called "direct reclaim." 

Direct reclaim normally works, and it is a good way to throttle memory-hungry processes, but it also suffers from a couple of significant problems. One of those is stack overflows; direct reclaim can happen from almost anywhere in the kernel, so it may be that the kernel stack is already mostly used before the reclaim process even starts. But if reclaim involves writing file pages back, it can be just the beginning of a long call chain in its own right, leading to the overflow of the kernel stack. Beyond that, direct reclaim, which reclaims pages wherever it can find them, tends to create seek-intensive I/O, hurting the whole system's I/O performance. 

Both of these problems have been seen on production systems. In response, a number of filesystems have been changed so that they simply ignore writeback requests which come from the direct reclaim code. That makes the problem go away, but it is a kind of papering-over that pleases nobody; it also arguably increases the risk that the system could go into the dreaded out-of-memory state. 

Mel Gorman has been working on the reclaim problem, on and off, for a few months now. His [latest patch set](<http://lwn.net/Articles/396512/>) will, with luck, improve the situation. The actual changes made are relatively small, but they apparently tweak things in the right direction. 

The key to solving a problem is understanding it. So, perhaps, it's not surprising that the bulk of the changes do not actually affect writeback; they are, instead, tracing instrumentation and tools which provide information on what the reclaim code is actually doing. The new tracepoints provide visibility into the nature of the problem and, importantly, how much each specific change helps. 

The core change is deep within the direct reclaim loop. If direct reclaim stumbles across a page which is dirty, it now must think a bit harder about what to do with it. If the dirty page is an anonymous (process data) page, writeback happens as before. The reasoning here seems to be that the writeback path for these pages (which will be going to a swap area) will be simpler than it is for file-backed pages; there are also fewer opportunities for anonymous pages to be written back via other paths. As a result, anonymous writeback might still create seek problems - but only if the swap area shares a spindle with other, high-bandwidth data. 

For dirty, file-backed pages, the situation is a little different; direct reclaim will no longer try to write back those pages directly. Instead, it creates a list of the dirty pages it encounters, then hands them over to the appropriate background process for the real writeback work. In some cases (such as when [lumpy reclaim](<http://lwn.net/Articles/211505/>) is trying to free specific larger chunks of memory), the direct reclaim code will wait in the hope that the identified pages will soon become free. The rest of the time, it simply moves on, trying to find free pages elsewhere. 

Handing the reclaim work over to the threads which exist for that task has a couple of benefits. It is, in effect, a simple way of switching to another kernel stack - one which is known to be mostly empty - before heading into the writeback paths. Switching stacks directly in the direct reclaim code had been discussed, but it was decided that the mechanism the kernel already has for switching stacks (context switches) was probably the right thing to use in this situation. Keeping the writeback work in kswapd and the per-BDI writeback threads should also help performance, since those threads try to order operations to minimize head seeks. 

When this problem was discussed in April, Andrew Morton pointed out that, over time, the amount of memory written back in direct reclaim has grown significantly, with an adverse effect on system performance. He wanted to see thought put into why that change has happened rather than trying to mitigate its effects. The final patch in Mel's series looks like an attempt to address this concern. It changes the direct reclaim code so that, if that code starts encountering dirty pages, it pokes the writeback threads and tells them to start cleaning pages more aggressively. The idea here is to keep the normal reclaim mechanisms running at a fast-enough pace that direct reclaim is not needed so often. 

This tweak seems to have a significant effect on some benchmarks; Mel says: 

Apparently, background flush must have been doing a better job getting [pages] cleaned in time and the direct reclaim stalls are harmful overall. Waking background threads for dirty pages made a very large difference to the number of pages written back. With all patches applied, just 759 filesystem pages were written back in comparison to 581811 in the vanilla kernel and overall the number of pages scanned was reduced. 

Anybody who likes digging through benchmark results is advised to look at Mel's patch posting - he appears to have run just about every test that he could to quantify the effects of this patch series. This kind of extensive benchmarking makes sense for deep memory management changes, since even small changes can have surprising results on specific workloads. At this point, it seems that the changes have the desired effect and most of the concerns expressed with previous versions have been addressed. The writeback changes, perhaps, are getting ready for production use.
