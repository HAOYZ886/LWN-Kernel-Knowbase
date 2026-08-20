---
title: Approaches to reducing TLB pressure
url: https://lwn.net/Articles/1016009/
date: "April 2, 2025"
category: "Memory management-Translation lookaside buffer"
author: "By Jonathan Corbet April 2, 2025 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 2, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The CPU's translation lookaside buffer (TLB) caches the results of virtual-address translations, significantly speeding memory accesses. TLB misses are expensive, so a lot of thought goes into using the TLB as efficiently as possible. Reducing pressure on the TLB was the topic of Rik van Riel's memory-management-track session at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit. Some approaches were considered, but the session was short on firm conclusions. 

Whenever the kernel changes a virtual-address mapping, it must take care to remove any TLB entries referring to the old mapping, or memory accesses could end up going to the wrong place. This TLB invalidation can be an expensive operation, since it can require sending inter-processor interrupts (IPIs) to every CPU on the system. Van Riel started his session by saying that he has been working on optimizing TLB invalidation on AMD CPUs by using a special-purpose instruction that eliminates the need for IPIs; [that work](<https://lwn.net/ml/all/20250226030129.530345-1-riel@surriel.com/>) has since been merged for the 6.15 release. 

[![\[Rik van Riel\]](https://static.lwn.net/images/conf/2025/lsfmm/RikvanRiel-sm.png)](<https://lwn.net/Articles/1016010/>) He also has been [working](<https://lwn.net/ml/all/20250319132818.1003878b@fangorn/>) to reduce the number of TLB flushes required when pages are reclaimed. He expected to see a nice performance improvement, but this optimization turned out to have almost no impact at all. Looking more closely at the workload in question, he found that it was experiencing about two-million TLB misses per second — on each CPU. This was a system with 200 CPUs. So reducing the number of TLB invalidations by a few thousand just did not help much. 

So he started looking at ways to increase the use of transparent huge pages. One of the advantages of huge pages is that an entire huge page can be referenced by a single TLB entry, greatly increasing the amount of address space that can be covered by the TLB. But the memory-management subsystem is oriented toward PMD-level huge pages, which are 2MB in size on many systems; they create more memory pressure as the result of internal fragmentation, outweighing the savings that they provide. More recent AMD and Arm CPUs, though, can coalesce TLB entries for smaller groups of physically contiguous pages. If the kernel could use more multi-size transparent huge pages (mTHPs), it could take better advantage of this feature and get better TLB utilization while reducing internal fragmentation. 

Unfortunately, he said, the kernel's existing mechanisms for managing transparent huge pages — the `khugepaged` thread and the huge-page shrinker — cannot currently do much with mTHPs. Relying on luck to create mTHPs will not work, especially on Arm CPUs, where explicit page-table-entry bits must be set to enable coalescing. So the kernel needs to learn to create mTHPs and, when necessary, split larger huge pages into mTHPs. 

There is [ongoing work](<https://lwn.net/Articles/1009039/>) to address these concerns; it was raised during the session but not discussed at length. Some time was spent on the "`max_pte_none`" problem (described at length in the linked article) that determines how many contiguous pages must be used before they can be coalesced into a huge page. 

Shakeel Butt asked if there is a way for user space to help with this problem. Van Riel answered that developers could look at switching to different allocation sizes; putting allocations closer together would also help to reduce TLB misses. But this is a hard problem, since functions like `malloc()` do not know how higher-level code will be using the allocated memory. 

A member of the audience pointed out that AMD's ability to perform TLB coalescing without explicit page-table-entry bits is nice, but it only works if the application has accessed at least half of the pages within the mTHP. The wider conclusion to draw from this detail is that techniques for mTHP management that work on one architecture, or even one CPU generation, may not work on others. 

Matthew Wilcox said that the biggest benefit from mTHP use is reducing the size of the kernel's least-recently-used (LRU) lists. That leads to lower lock-hold times when performing reclaim, and much less contention overall. So, even in the absence of TLB-utilization improvements, using larger pages is worth the trouble. 

The final part of the discussion touched on a number of details related to the management of mTHPs. Ryan Roberts pointed out that if an application calls [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) on a portion of an mTHP, the entire mTHP must be flushed from the TLB, which is costly. In such cases, it may have been better to not create the mTHP in the first place. Van Riel suggested that the kernel could watch what user space is doing and react accordingly if it sees a lot of `madvise()` calls or unused pages within mTHPs. Roberts added that the kernel's copy-on-write (COW) mechanism works on individual pages, causing mTHPs to be split; it would be good to investigate performing COW on larger folios. Similarly, swapping can split folios which, as van Riel added, is not optimal. Compressed swapping mechanisms like zswap are more effective when applied to larger chunks of memory. 

As time ran out, David Hildenbrand pointed out that, for AMD CPUs, the page-table-entries of a range of pages must match for TLB coalescing to happen. Perhaps, he suggested, more thought needs to be put into how those bits are set to avoid blocking coalescing inadvertently.
