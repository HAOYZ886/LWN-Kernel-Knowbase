---
title: "LSFMM: Improving the swap subsystem"
url: https://lwn.net/Articles/548118/
date: "April 23, 2013"
category: "Memory management-Swapping"
author: "By Jonathan Corbet April 23, 2013 LSFMM Summit 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 23, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Huadong Liu led a memory-management-track session on the performance of the Linux swap subsystem. The discussion almost certainly did not go the way he intended, but it may have some interesting results for future swap development. 

Huadong started with a complaint: he had been running some tests using a solid-state drive as a swap device and was not getting the performance that he expected. There was, he said, too much lock contention, too much TLB flushing overhead, and, perhaps most significantly, a scattered I/O pattern resulting from multiple processes running in direct reclaim at the same time. He also suggested that readahead on page faults was impacting performance for workloads with random access patterns. 

Where things went a little off the rails, though, was when Huadong said that the system was reclaiming page-cache pages far too aggressively. When he set the "swappiness" parameter ([described here](<https://lwn.net/Articles/83588/>) in 2004) to 200 (causing the system to reclaim swap-backed pages in preference to file-backed pages), performance improved considerably. Hugh Dickins asked why Huadong felt the need to change the swapping balance in that way. Various possible reasons were passed back and forth, but the reality appeared to be this: the system was using a solid-state drive for swap, but everything else was on rotating storage. So, naturally, swap operations (involving anonymous pages) would be faster than page-cache operations. On a fully solid-state system, the results would be likely to differ considerably. 

That was the jumping-off point for a new proposal from Hugh, who claimed that the memory management subsystem — and swap in particular — should be much more responsive to the relative speeds of the devices used for backing store. Currently all such devices are treated equally, but, increasingly, they are anything but equal. The system would perform a lot better if it could, for example, make a point of using faster devices before resorting to the slower ones. That was, he said, a lesson that could be learned from Huadong's experience. 

Making proper use of information about backing-device speeds, Hugh said, would require a fundamental change to how swapping is handled in the kernel. Currently, when a page is swapped out, its location is stored in the page table entry (PTE). That is problematic because it requires that location to be chosen quite early and makes it impossible to change thereafter. If the location of the swapped page on disk could be chosen at the last moment, it would be easier to direct pages to the fastest devices; the swap code could also place outgoing pages next to each other, allowing the block layer to merge the resulting I/O requests. 

Getting there would require, inevitably, the addition of another layer of indirection. The "swap location" stored in the PTE would be an index into a table where the real location could be found. That location would not be assigned until the time came to actually write the page to swap, allowing the decision to be deferred until the last moment. This mechanism would also allow swapped pages to be relocated if desired; a little-used page on a fast device could eventually be migrated to slower storage, for example. 

Hugh claimed that this bit of work should be done first; that failure to add the necessary infrastructure could impede the improvement of the swap subsystem in general. Mel Gorman worried, though, that basing swap decisions on the speed of the backing store device would make the system less predictable and could fall apart on some workloads. Andi Kleen suggested using more huge pages; performing swap in large batches would make a lot of the pain go away. Rik van Riel worried that using huge pages would lead to internal fragmentation and more I/O bandwidth consumption, but Andi responded that bandwidth is not the limiting factor — the number of I/O operations that can be performed is the bottleneck. 

In the end, the consensus seemed to be that Hugh's plan for deferred assignment of swap location made sense. We may see an implementation posted in the not-too-distant future. The session ended with Hugh apologizing for "hijacking" the discussion, but nobody seemed to upset about the direction things had taken.
