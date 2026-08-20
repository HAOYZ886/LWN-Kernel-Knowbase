---
title: Huge page issues
url: https://lwn.net/Articles/592011/
date: "March 26, 2014"
category: "Huge pages; Memory management-Huge pages"
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

Using huge pages can improve performance on a number of workloads, mostly through decreased paging costs and better translation lookaside buffer usage. But supporting huge pages imposes costs of its own on the kernel. The memory management track of the 2014 Linux Storage, Filesystem, and Memory Management Summit set aside some time to talk about those costs and how they might be reduced. 

Aneesh Kumar started out by saying that, on architectures with larger page sizes (such as PowerPC), users tend to disable the transparent huge pages feature because it causes performance problems. Those problems might result from the fact that, when normal pages are larger (64KB, for example), huge pages are also larger. They may be getting large enough that internal fragmentation is a concern. One possible solution might be to split huge pages when they are swapped out of memory. That might reduce the amount of I/O required, especially if pages filled with zeroes can be skipped during the swapout process. There might also be some gains to be had by disabling the allocation of huge pages during page fault time, leaving it to the khugepaged daemon to assemble huge pages later on. 

A question arises on NUMA systems: if a huge page cannot be allocated locally, is it better to allocate a remote huge page or a local small page? The benefits from using remote huge pages seemingly do not outweigh the costs of doing so. There was some confusion on this issue; Rik van Riel thought that falling back to small pages locally should already be the default course of action. Matthew Wilcox suggested that, if a page must be allocated remotely, it should always be a small page; huge pages should only be allocated on the local node. 

There was also some talk of adding a new heuristic that would disable the allocation of huge pages in situations where memory is highly fragmented. When fragmentation happens, it might well be better to try to reduce overall memory usage by sticking with small pages. Once again, khugepaged can collapse things into huge pages later if the resources become available. 

Peter Zijlstra had a different suggestion: take the transparent huge page mechanism out of the kernel entirely and leave the associated headaches behind. Andrew Morton agreed that transparent huge pages have "made a mess of the kernel." Hugh Dickins expanded on that thought, noting that the memory management subsystem as a whole has gotten significantly more complex, and that the transparent huge page feature is a big part of the problem. It is a feature that does benefit some workloads, and, he said, it was a magnificent technical achievement. But the way transparent huge pages rely on memory compaction, extensive reference counting, and complex code are downsides to the feature. 

Rik noted that much of the complexity comes from the feature having been retrofitted onto the existing memory management code; it may be time to look at simplification through extensive rewriting of the code. Andrew agreed with a focus on simplicity, stating that the memory management code has gone beyond the developers' ability to maintain it. 

Davidlohr Bueso shifted the conversation a bit, noting that HPUX has a "zero page daemon" charged with zero-filling huge pages when the CPU is otherwise idle. He would like to add a similar feature to Linux. It would work with the hugetlbfs subsystem only; transparent huge pages would not be zeroed in this way. Working with hugetlbfs is beneficial in that the kernel knows just how many pages have been configured, so there is an automatic bound on how many pages would need to be zeroed. 

The benefit of such a mechanism would mainly be in reduced application-startup time. But it is unlikely to find its way into the mainline; Mel stated firmly that he saw it as a bunch of additional code for little real benefit. Adding work to the idle loop would have power-consumption implications, increase memory bandwidth usage, and lead to confusing variability in application performance. It is, he thought, a poor tradeoff overall. Andrew suggested finding a way to do the page zeroing from user space; that would make the overhead visible and put it all under user control. 

Transparent huge pages currently only work with anonymous pages; file-backed pages are not covered. Kirill Shutemov talked about his work to extend transparent huge pages to the page cache (described briefly in [this article](<https://lwn.net/Articles/562211/>)). The work is about one year old and has shown significant improvements on some benchmarks. This improvement has come at the cost of adding a new lock to protect the page cache registry. Things get especially complicated when huge pages need to be split. 

At that point, the discussion headed into the question of whether it is ever really necessary to split a huge page in the page cache. With anonymous pages, there are times when splitting is nearly unavoidable; performing copy-on-write on a page within a huge page is one example. But the page cache works differently. It might well be possible for some processes to map parts of a huge page as small pages while others see it as a single huge page cache page. But there are some interesting reference counting questions to be answered. 

Reference counts for page cache pages live in the associated `struct page` in the system memory map. When the page is a huge page, there are many `page` structures, one for each small page that goes into the huge page. A reference count for a huge page is kept in the first of those `page` structures, while the reference count in the remaining "tail pages" are not used. But if some threads see only a few of the tail pages rather than the huge page as a whole, where should the reference count for those tail pages be? Rik suggested that the head page should be used to maintain a reference count for the whole huge page; a reference to the huge page would then set the count to 512 (the number of small pages that fit into a huge page). If the page is split, references to individual small pages can be dropped by decrementing the head-page counter, and the kernel still knows how many references to the huge page (or parts thereof) still exist. 

Hugh worried that there could be confusion between the head page as representing a huge page and that page on its own as a small page, but Rik thought those issues could be worked out. 

Should this work go upstream? Andrew suggested it should if, in the end, it makes things simpler. He also said that, in retrospect, the memory management subsystem should have been designed around variable page sizes from the beginning, but nobody was thinking in those terms. Any future work should be done with that kind of end goal in mind, though; grafting features like huge pages onto the existing memory management code has clearly been a mistake. That, of course, is a tall order for anybody wanting to improve the kernel's management of huge pages; it suggests that we could be seeing some fundamental changes in memory management in the coming years. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
