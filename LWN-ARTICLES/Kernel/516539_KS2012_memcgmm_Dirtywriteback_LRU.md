---
title: "KS2012: memcg/mm: Dirty/writeback LRU"
url: https://lwn.net/Articles/516539/
date: "September 17, 2012"
category: "Memory management-Writeback"
author: "By Michael Kerrisk September 17, 2012 2012 Kernel Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Michael Kerrisk**  
September 17, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

During the 2012 Kernel Summit [memcg/mm minisummit](<https://lwn.net/Articles/516439/>), Fengguang Wu discussed an adjustment to the current LRU implementation, whereby the LRU (least recently used) list is [split into anonymous and file LRU lists](<http://linux-mm.org/PageReplacementDesign>). Fengguang has a proposal to split the file LRU list into clean and dirty lists. The proposed idea would not necessarily enforce 100% clean or dirty pages on the lists, but that is a minor detail that does not affect his goal. The general objective is that, in the presence of a process that is dirtying a large number of pages, other processes do not pay a high cost for scanning dirty pages on the list. This would be particularly true for cgroups, where it can happen that 100% of the inactive list is dirty pages. 

There was not much support in the room for splitting the LRU in this manner, since doing so loses aging information. There are already problems selecting the ratio for scanning anonymous pages and file-backed pages and some developers felt that adding yet another LRU list would compound the problem. It would require very compelling evidence to merge such a feature. 

However, there was support for adding a discard list to which pages marked `PageReclaim` get moved. (If the page-reclaim algorithm encounters a page at the end of the LRU that is currently being written to disk, then it marks it `PageReclaim` instead of waiting on the I/O to complete. When the I/O completes, the page is now considered clean and is immediately discarded. For the curious, the number of pages that are treated this way is recorded in the `nr_vmscan_immediate_reclaim` counter in `/proc/vmstat`.) The pages would be scanned only once by reclaim and the aging information would no longer be relevant, since the pages are going to be reclaimed immediately when writeback completes. 

A [patch](<https://lwn.net/Articles/516436/>) exists that implements this separate LRU list. Unfortunately the patch is buggy, and gets the accounting of how many pages are on each LRU incorrect. To properly implement it would require a page flag, but there are currently no free flags available. The feature is not considered important enough to justify increasing the size of the page-flags bit mask to 64 bits. Johannes Weiner is going to check to see if he can free up one of the existing page flags, so as to allow this patch to be finished and merged. 

[Next: Proportional I/O controller](<https://lwn.net/Articles/516540/>)
