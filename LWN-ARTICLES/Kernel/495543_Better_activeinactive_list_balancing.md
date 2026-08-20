---
title: Better active/inactive list balancing
url: https://lwn.net/Articles/495543/
date: "May 2, 2012"
category: "Memory management-Page replacement algorithms"
author: "By Jonathan Corbet May 2, 2012"
---

By **Jonathan Corbet**  
May 2, 2012

Memory management is a notoriously tricky task, though the underlying objective is quite clear: look into the future and ensure that the pages that will be needed by applications are in memory. Unfortunately, existing crystal ball peripherals tend not to work very well; they also usually require proprietary drivers. So the kernel is stuck with a set of heuristics that try to guess future needs based on recent behavior. Adjusting those heuristics is always a bit of a challenge; it is easy to put in changes that will break obscure workloads years in the future. But that doesn't stop developers from trying. 

A core part of the kernel's memory management subsystem is a pair of lists called the "active" and "inactive" lists. The active list contains anonymous and file-backed pages that are thought (by the kernel) to be in active use by some process on the system. The inactive list, instead, contains pages that the kernel thinks might not be in use. When active pages are considered for eviction, they are first moved to the inactive list and unmapped from the address space of the process(es) using them. Thus, once a page moves to the inactive list, any attempt to reference it will generate a page fault; this "soft fault" will cause the page to be moved back to the active list. Pages that sit in the inactive list for long enough are eventually removed from the list and evicted from memory entirely. 

One could think of the inactive list as a sort of probational status for pages that kernel isn't sure are worth keeping. Pages can get there from the active list as described above, but there's another way to inactive status as well: file-backed pages, when they are faulted in, are placed in the inactive list. It is quite common that a process will only access a file's contents once; requiring a second access before moving file-backed pages to the active list lets the kernel get rid of single-use data relatively quickly. 

Splitting memory into two pools in this manner leads to an immediate policy decision: how big should each list be? A very large inactive list gives pages a long time to be referenced before being evicted; that can reduce the number of pages kicked out of memory only to be read back in shortly thereafter. But a large inactive list comes at the cost of a smaller active list; that can slow down the system as a whole by causing lots of soft page faults for data that's already in memory. So, as is the case with many memory management decisions, regulating the relative sizes of the two lists is a balancing act. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The way that balancing is done in current kernels is relatively straightforward: the active list is not allowed to grow larger than the inactive list. Johannes Weiner has concluded that this heuristic is too simple and insufficiently adaptive, so he has come up with [a proposal for a replacement](<https://lwn.net/Articles/495423/>). In short, Johannes wants to make the system more flexible by tracking how long evicted pages stay out of memory before being faulted back in. 

Doing so requires some significant changes to the kernel's page-tracking infrastructure. Currently, when a page is removed from the inactive list and evicted from memory, the kernel simply forgets about it; that clearly will not do if the kernel is to try to track how long the page remains out of memory. The page cache is tracked via a [radix tree](<https://lwn.net/Articles/175432/>); the kernel's radix tree implementation already has a concept of "exceptional entries" that is used to track tmpfs pages while they are swapped out. Johannes's patch extends this mechanism to store "shadow" entries for evicted pages, providing the needed long-term record-keeping for those pages. 

What goes into those shadow entries is a representation of the time the page was swapped out. That time can be thought of as a counter of removals from the inactive list; it is represented as an `atomic_t` variable called `workingset_time`. Every time a page is removed from the inactive list, either to evict it or to activate it, `workingset_time` is incremented by one. When a page is evicted, the current value of `workingset_time` is stored in its associated shadow entry. This time, thus, can be thought of as a sort of sequence counter for memory management events. 

If and when that page is faulted back in, the difference between the current `workingset_time` and the value in the shadow entry gives a count of how many pages were removed from the inactive list while that page was out of memory. In the language of Johannes's patch, this difference is called the "refault distance." The observation at the core of this patch set is that, if a page returns to memory with a refault distance of _R_ , its eviction and refaulting would have been avoided had the inactive list been _R_ pages longer. _R_  is thus a sort of metric describing how much longer the inactive list should be made to avoid a particular page fault. 

Given that number, one has to decide how it should be used. The algorithm used in Johannes's patch is simple: if _R_ is less than the length of the active list, one page will be moved from the active to the inactive list. That shortens the active list by one entry and places the formerly-active page on the inactive list immediately next to the page that was just refaulted in (which, as described above, goes onto the inactive list until a second access occurs). If the formerly-active page is still needed, it will be reactivated in short order. If, instead, the working set is shifting toward a new set of pages, the refaulted page may be activated instead, taking the other page's place. Either way, it is hoped, the kernel will do a better job of keeping the right pages active. Meanwhile, the inactive list gets slightly longer in the hope of avoiding refaults in the near future. 

How well all of this works is not yet clear: Johannes has not posted any benchmark results for any sort of workload. This is early-stage work at this point, a long way from acceptance into a mainline kernel release. So it could evolve significantly or fade away entirely. But more sophisticated balancing between the active and inactive lists seems like an idea whose time may be coming.
