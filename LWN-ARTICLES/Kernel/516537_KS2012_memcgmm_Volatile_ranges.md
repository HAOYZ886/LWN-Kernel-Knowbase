---
title: "KS2012: memcg/mm: Volatile ranges"
url: https://lwn.net/Articles/516537/
date: "September 17, 2012"
category: Volatile ranges
author: "By Michael Kerrisk September 17, 2012 2012 Kernel Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Michael Kerrisk**  
September 17, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

John Stultz kicked off a session of the 2012 Kernel Summit [memcg/mm minisummit](<https://lwn.net/Articles/516439/>) by saying that his work on [volatile ranges](<https://lwn.net/Articles/500382/>) was motivated by Android's `ashmem`, and he thanked Robert Love specifically. He then proceeded to explain the basic idea of volatile ranges, which is to provide a means whereby a process can inform the kernel that it can discard specific pages if memory pressure is high; if the pages are discarded, then it is assumed that the process has sufficient information that it can re-create the data. He covered the history of the different approaches that he tried in the past and what problems he had with each of them. 

The current approach he is considering is to use an LRU (least recently used) list of "easily reclaimable" pages that at a minimum would contain all the pages that have been marked volatile. This new LRU list would always be shrunk first when there is memory pressure and pages must be reclaimed. Andrea Arcangeli argued that this should be unnecessary and that the pages should instead just be moved to the tail of the anonymous page LRUs in order to be reclaimed first; but others pointed out that if there is no swap, then the inactive list for anonymous pages does not rotate. Rik van Riel felt that the "volatile ranges" pages should not necessarily always be reclaimed first and the decision about whether to reclaim from them should come down to the cost of re-creating the data. However, the kernel can't know that cost, so the point could be argued either way. 

The discussion moved on to the cost of moving pages between LRUs and detecting when the data needs to be re-created. Having declared a range of pages as volatile, an application can later declare them non-volatile (if it decides it needs to access them again); for the latter operation, the application needs to use the system-call return value to detect when the data needs to be recovered. Continually marking pages volatile and non-volatile like this would be very expensive, but not enough people were familiar enough with John's patch set to give concrete proposals on how the implementation could be improved. A few ideas were bounced around, but John appealed to memory-management developers to look more closely at his patches and give advice on how the implementation should be finished, because he feels he is insufficiently experienced with memory management to make all the decisions. 

The discussion also covered a few other points. John was asked what the semantics are if a process accesses a page that has been marked as volatile. Currently, he said, you get either data (if the page has not been discarded) or zeros (if it has). Maybe, he said, we should deliver a `SIGSEGV` signal; Hugh Dickins suggested that `SIGBUS` would be better. The current user-space interface is `fallocate()`, and the question was raised as to whether this was the right API. Hugh Dickins felt that `fadvise()` or `madvise()` would be more appropriate. (However, things are going a little bit in circles here, since an earlier version of John's patches used `fadvise()`, but Dave Chinner [suggested](<https://lwn.net/Articles/516434/>) that `fallocate()` would be better suited.) 

[Next: Moving zcache towards the mainline](<https://lwn.net/Articles/516538/>)
