---
title: Improvements for the contiguous memory allocator
url: https://lwn.net/Articles/1016844/
date: "April 16, 2025"
category: "Contiguous memory allocator; Memory management-Large allocations"
author: "By Jonathan Corbet April 16, 2025 LSFMM+BPF"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 16, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

As a system runs, its memory becomes fragmented; it does not take long before the allocation of large, physically contiguous memory ranges becomes difficult or impossible. The [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>) is a kernel subsystem that attempts to address this problem, but it has never worked as well as some would like. Two sessions in the memory-management track at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit looked at how CMA can be improved; the first looked at providing guaranteed allocations, while the second addressed some inefficiencies in CMA. 

In current kernels, CMA works by reserving a large area of physically contiguous memory at boot time; it uses that area to satisfy requests for large allocations later on. In the absence of such allocations, the CMA reservation can be allocated for other purposes, but only movable allocations can be placed there. The intent is that, should a need for a large buffer arise, those other allocations can be migrated elsewhere in memory, freeing a physically contiguous range. The hoped-for result is that large allocations are always possible, but that the memory used for those allocations is not wasted when it is not needed. 

#### Guaranteed CMA

Suren Baghdasaryan ran a session to discuss the [guaranteed contiguous memory allocator](<https://lwn.net/Articles/1015000/>) patches that he had posted shortly before the conference. This series, he said, was mostly based on older work from Minchan Kim and SeongJae Park, along with the "cleancache" abstraction first [proposed](<https://lwn.net/Articles/545244/>) by Dan Magenheimer in 2012. 

[![\[Suren
Baghdasaryan\]](https://static.lwn.net/images/conf/2025/lsfmm/SurenBaghdasaryan-sm.png)](<https://lwn.net/Articles/1016854/>) The kernel has long provided two options for physically contiguous allocations, he said. The first is CMA, which has the advantage that the reserved memory can be used for movable allocations when it is not needed. The downside of CMA is that, sometimes, as was [discussed two days earlier](<https://lwn.net/Articles/1015551/>), those "movable" allocations prove not to be movable after all; that can cause CMA allocations to fail. Even when allocations succeed, the time required is nondeterministic, since how long it takes to move pages out of the way can vary widely. Both of these problems can be serious. The Android face-unlock application needs to be able to allocate a buffer from CMA; since a user is waiting to access their device, the application cannot tolerate slow allocations or the possibility of not getting its memory at all. 

The alternative is carve-outs — setting aside memory at boot but not allowing it to be used for any other purpose. Carve-outs are guaranteed to work quickly, but they also waste memory to provide a buffer that may almost never be used. 

To create a better solution, Baghdasaryan set a rule that the CMA area can only be used to cache recoverable content that can be dropped immediately on demand. That memory should otherwise be inaccessible. It can be used to cache useful data, but not to hold anything that might impede its immediate use when a large buffer is needed. 

The tool he used to implement his solution is cleancache. As with the original cleancache design, guaranteed CMA will store clean, file-backed pages that can be dropped on demand. It becomes a sort of extension to the page cache that can avoid the need to perform I/O when reclaimed pages are faulted back in. In the new implementation, though, the invasive filesystem hooks required by the original are gone; instead, there are some simple hooks in the memory-management subsystem's fault, eviction, and invalidation paths. It has a simple API that allows pages to be donated to the cache and gotten back quickly when they are needed. 

Guaranteed CMA is a backend for cleancache; like CMA, it reserves its region at boot time. The existing devicetree entries used to configure CMA now work for the new version as well. The reserved region is donated to the cleancache, where it can be used until needed. The result is not perfect utilization of the reserved memory; in Baghdasaryan's tests, about 40% of that memory was holding cached data at any given time. The hit rate for queries to the cache was 42%, though, indicating that a lot of I/O had been avoided. 

David Hildenbrand asked whether the biggest problem with CMA as it exists now is the latency on allocation requests, or the possibility of allocation failures; Baghdasaryan said that both were big problems for the face-unlock case. Hildenbrand asked what the cause of the failures was, surmising that it might be pages that have been pinned by other subsystems. Baghdasaryan said that might be the source of the problem; direct I/O or writeback might also play into it. Hildenbrand said it would be better to make existing CMA more reliable if possible; Baghdasaryan's approach is good, he said, but may not work well on systems where the workload is dominated by anonymous pages (which cannot be dropped, and so cannot be stored in the cache). 

After acknowledging that suggestion, Baghdasaryan concluded with a few loose ends. There may be other uses for guaranteed CMA, which could perhaps manage memory that is reserved for crash-dump kernels, for example. He wondered about security and, specifically, when pages stored in the cache should be zeroed. A bad kernel module could snoop around in the cache, he said, but bad modules can already do that now. He also wondered if guaranteed CMA needs some sort of NUMA awareness; Michal Hocko suggested keeping the implementation simple until a need for more complexity makes itself clear. 

As the session closed, Brendan Jackman asked whether this feature could be made available to user space. Baghdasaryan said that DMA buffers could perhaps live in the CMA region, but Jackman was interested in more generalized user-space caching. Hildenbrand suggested that regions mapped with `MAP_DROPPABLE` (which have similar "can be dropped at any time" semantics) could perhaps be stored in the CMA area. 

#### Optimizing CMA layout

[![\[Juan
Yescas\]](https://static.lwn.net/images/conf/2025/lsfmm/JuanYescas-sm.png)](<https://lwn.net/Articles/1016855/>) Later that day, Juan Yescas ran a session on a problem he has been experiencing with CMA. In short, on systems with large page sizes, huge areas have to be dedicated to CMA to be able to use it. Currently, the CMA region must be aligned to the kernel's page-block size which, in turn, is driven by an number of parameters, including the system's base-page size. The size of the region must also be a multiple of the page-block size. On systems with 64KB pages and transparent huge pages enabled, the page-block size can be 512MB; that becomes the effective minimum size of the CMA region. If only a fraction of that space is needed, the rest is set aside needlessly. 

Yescas was wondering why this alignment requirement exists. Hildenbrand pointed out that CMA has to work on page blocks to be able to migrate pages out when they are in the way of an allocation. 

Yescas had some solutions to the problem that he has been exploring. First would be to sum up all of the CMA requirements for the system, then allocate a single CMA region to hold all of them. That does not help, though, on systems with a single CMA user. An alternative is to set the `ARCH_FORCE_MAX_ORDER` configuration parameter to a smaller value like seven. That will result in smaller page blocks, minimizing the memory waste, but it also makes the allocation of huge pages harder. Finally, one could have CMA just allocate its memory from the kernel's buddy allocator when the reservation size is relatively small. 

Vlastimil Babka said that the CMA reservation is not really wasted, even if it exceeds the required size, because movable allocations can be placed there. Hildenbrand said that, in the long term, work should be done to ensure that page blocks are reasonable in size; there could be some sort of "super blocks" abstraction for larger groupings if needed. That is not an option for now, though. He suggested that Yescas could use guaranteed CMA, which does not use migration and, thus, does not need page-block alignment. 

On the other hand, Hildenbrand said at the end of the session, setting `ARCH_FORCE_MAX_ORDER` to a smaller value is not a good idea. Instead, it would be better to find a way to let the page-block size be smaller, as is apparently done with the PowerPC architecture now. That, he concluded, might be the cleanest short-term solution to the problem.
