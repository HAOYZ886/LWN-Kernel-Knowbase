---
title: "Avoiding - and fixing - memory fragmentation"
url: https://lwn.net/Articles/211505/
date: "November 28, 2006"
category: "Lumpy reclaim; Memory management-Large allocations"
author: 
---

Memory fragmentation is a kernel programming issue with a long history. As a system runs, pages are allocated for a variety of tasks with the result that memory fragments over time. A busy system with a long uptime may have very few blocks of pages which are physically-contiguous. Since Linux is a virtual memory system, fragmentation normally is not a problem; physically scattered memory can be made virtually contiguous by way of the page tables. 

But there are a few situations where physically-contiguous memory is absolutely required. These include large kernel data structures (except those created with `vmalloc()`) and any memory which must appear contiguous to peripheral devices. DMA buffers for low-end devices (those which cannot do scatter/gather I/O) are a classic example. If a large ("high order") block of memory is not available when needed, something will fail and yet another user will start to consider switching to BSD. 

Over the years, a number of approaches to the memory fragmentation problem have been considered, but none have been merged. Adding any sort of overhead to the core memory management code tends to be a hard sell. But this resistance does not mean that people stop trying. One of the most persistent in this area has been Mel Gorman, who has been working on an anti-fragmentation patch set for some years. Mel is back with [version 27 of his patch](<http://lwn.net/Articles/211194/>), now rebranded "page clustering." This version appears to have attracted some interest, and may yet get into the mainline. 

The core observation in Mel's patch set remains that some types of memory are more easily reclaimed than others. A page which is backed up on a filesystem somewhere can be readily discarded and reused, for example, while a page holding a process's task structure is pretty well nailed down. One stubborn page is all it takes to keep an entire large block of memory from being consolidated and reused as a physically-contiguous whole. But if all of the easily-reclaimable pages could be kept together, with the non-reclaimable pages grouped into a separate region of memory, it should be much easier to create larger blocks of free memory. 

So Mel's patch divides each memory zone into three types of blocks: non-reclaimable, easily reclaimable, and movable. The "movable" type is a new feature in this patch set; it is used for pages which can be easily shifted elsewhere using the kernel's [page migration](<http://lwn.net/Articles/160201/>) mechanism. In many cases, moving a page might be easier than reclaiming it, since there is no need to involve a backing store device. Grouping pages in this way should also make the creation of larger blocks "just happen" when a process is migrated from one NUMA node to another. 

So, in this patch, movable pages (those marked with `__GFP_MOVABLE`) are generally those belonging to user-space processes. Moving a user-space page is just a matter of copying the data and changing the page table entry, so it is a relatively easy thing to do. Reclaimable pages (`__GFP_RECLAIMABLE`), instead, usually belong to the kernel. They are either allocations which are expected to be short-lived (some kinds of DMA buffers, for example, which only exist for the duration of an I/O operation) or can be discarded if needed (various types of caches). Everything else is expected to be hard to reclaim. 

By simply grouping different types of allocation in this way, Mel was able to get some pretty good results: 

In benchmarks and stress tests, we are finding that 80% of memory is available as contiguous blocks at the end of the test. To compare, a standard kernel was getting < 1% of memory as large pages on a desktop and about 8-12% of memory as large pages at the end of stress tests. 

Linus has, in the past, been generally opposed to efforts to reduce memory fragmentation. His [comments](<https://lwn.net/Articles/211515/>) this time around have been much more detail-oriented, however: should allocations be considered movable or non-movable by default? The answer would appear to be "non-movable," since somebody always has to make some effort to ensure that a specific allocation can be moved. Since the discussion is now happening at this level, some sort of fragmentation avoidance might just find its way into the kernel. 

A related approach to fragmentation is the [lumpy reclaim mechanism](<http://lwn.net/Articles/211199/>) posted by Andy Whitcroft but originally by Peter Zijlstra. Memory reclaim in Linux is normally done by way of a least-recently-used (LRU) list; the hope is that, if a page must be discarded, going after the least recently used page will minimize the chances of throwing out a page which will be needed soon. This mechanism will tend to free pages which are scattered randomly in the physical address space, however, making it hard to create larger blocks of free memory. 

The lumpy reclaim patch tries to address this problem by modifying the LRU algorithm slightly. When memory is needed, the next victim is chosen from the LRU list as before. The reclaim code then looks at the surrounding pages (enough of them to form a higher-order block) and tries to free them as well. If it succeeds, lumpy reclaim will quickly create a larger free block while reclaiming a minimal number of pages. 

Clearly, this approach will work better if the surrounding pages can be freed. As a result, it combines well with a clustering mechanism like Mel Gorman's. The distortion of the LRU approach could have performance implications, since the neighboring pages may be under heavy use when the lumpy reclaim code goes after them. In an attempt to minimize this effect, lumpy reclaim only happens when the kernel is having trouble satisfying a request for a larger block of memory. 

If - and when - these patches may be merged is yet to be seen. Core memory management patches tend to inspire a high level of caution; they can easily create chaos when exposed to real-world workloads. The problem doesn't go away by itself, however, so something is likely to happen, sooner or later.
