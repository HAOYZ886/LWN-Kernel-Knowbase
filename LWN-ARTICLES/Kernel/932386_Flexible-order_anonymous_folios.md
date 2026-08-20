---
title: "Flexible-order anonymous folios"
url: https://lwn.net/Articles/932386/
date: "May 25, 2023"
category: "Memory management-Folios"
author: "By Jonathan Corbet May 25, 2023 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 25, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The conversion to [folios](<https://lwn.net/Articles/849538/>) is intended to, among other things, make it easy for the kernel to manage chunks of memory in a number of different sizes. So far, though, that flexibility is not being used in the kernel's handling of anonymous pages. At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Yu Zhao and Yang Shi ran a session in the memory-management track aimed at charting a path toward support for anonymous pages in a variety of sizes. 

Zhao began by pointing out that the kernel currently only handles two sizes of anonymous pages — the base (usually 4KB) size and the PMD size (usually 2MB huge pages). Recent CPUs, though, have added support for the coalescing of translation-lookaside-buffer (TLB) entries, allowing a single entry to cover as many as four physically contiguous pages, often transparently. Taking advantage of that capability would improve performance on those CPUs. 

[![\[Yu Zhao\]](https://static.lwn.net/images/conf/2023/lsfmm/YuZhao-sm.png)](<https://lwn.net/Articles/932389/>) Managing memory in base-page units is not scalable, he said. It makes the system deal with large number of page faults, least-recently-used (LRU) lists containing millions of pages, and increased TLB-flushing costs. Using a larger base-page size can improve performance, but at the cost of increased internal fragmentation and, perhaps, forcing user-space changes (a recompile if nothing else). But mid-sized folios might just be "a sweet spot" for a number of reasons. Internal fragmentation is reduced, and the presence of an "accessed" bit for each base page means that sparsely used folios can be broken up. Larger folios are entirely transparent to user space, and easier than huge pages to allocate; the ability to use TLB coalescing will also make them perform better. 

Implementing larger anonymous folios will require solving some problems, though, starting with finding a suitable policy for when they should be used at all. Some heuristics, including looking at alignment and sizing, can be used to find suitable virtual memory areas (VMAs). Should a single large size be used, he asked, or should it vary from one VMA to the next? For an allocation policy, he suggested attempting to allocate huge pages first, then falling back to a folio at the TLB-coalescing size, then base pages if all else fails. 

Behavior under memory pressure is another thing that should be thought through, he said. But, even then, trying to allocate the largest sizes first is probably the best policy, as long as care is taken to avoid forcing excessive reclaim when the larger sizes are not available. Pasha Tatashin said that reclaim from larger folios could also be tricky, since freeing them might require first making another allocation. 

Shi asked whether the allocation process should try all of the page orders from the huge-page size on down, or if it should, instead, skip some orders. Matthew Wilcox said that, with the right allocation flags, trying all orders might be fine, but it might be better to modify the page allocator to return the largest size available up to a given order and avoid making multiple allocation calls. 

There was a brief digression into the proper use of the `mapcount` field of `struct page`. In theory it tracks the number of contexts that have the given page mapped, but the use of this field has [led to bugs](<https://lwn.net/Articles/895439/>) in the past and there is disagreement over what its semantics should be. Wilcox said that the use of `mapcount` for higher-order pages needs to be rethought. 

With a new virtual-memory feature comes a need for statistics to track it; at least one new counter will be needed to track large-folio use, but Zhao wondered if more would be required. Wilcox suggested just counting folios as single pages, but Tatashin said that would make it impossible for users to understand what was happening in their programs. Wilcox answered that they'll know that large folios are in use when their programs run faster. Tatashin said that users would want to be able to debug why their programs _aren't_ going faster; Wilcox suggested `/proc/_pid_ /smaps` or tracepoints, but Shi said additional counters would also be helpful there. 

Zhao discussed the reclaim process briefly, repeating that the base-page access bits can be used to detect internal fragmentation, where only part of a large folio is actually being used. A heuristic can be used to determine when a sparsely used folio should be split. He asked whether large folios should be swapped as a unit; the answer from the group was "yes". 

There were a few other complications to be rushed through as the session ran out of time. The memory-compaction code currently skips large folios; it needs to learn how to work with them as well. Collapsing individual pages into large folios could improve performance, but it has to be done carefully to keep `khugepaged` from working against the reclaim code. When a large page needs to be split, the question of whether to split all the way to base pages or to keep larger sizes arises. 

Zhao concluded by saying that [there is an RFC patch set](<https://lwn.net/ml/linux-mm/20230414130303.2345383-1-ryan.roberts@arm.com/>) implementing large anonymous folios in circulation, and that the group should have a look at it.
