---
title: Three ways to rework the swap subsystem
url: https://lwn.net/Articles/1016136/
date: "April 7, 2025"
category: "Memory management-Swapping"
author: "By Jonathan Corbet April 7, 2025 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 7, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The kernel's swap subsystem is complex and highly optimized — though not always optimized for today's workloads. In three adjacent sessions during the memory-management track of the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit, Kairui Song, Nhat Pham, and Usama Arif all talked about some of the problems that they are trying to solve in the Linux swap subsystem. In the first two cases, the solutions take the form of an additional layer of indirection in the kernel's swap map; the third, which enables swap-in of large folios, may or may not be worthwhile in the end. 

#### Simplifying the swap subsystem

There are some good things about the kernel's swapping code, Song began. Since swapping is done to make memory available to the kernel, it must use as little memory as possible itself; the swap code manages to get by with a single byte of overhead per page of swap space. Most swapping operations are fast and lightweight; swap entries point directly to physical locations. The per-CPU swap-cluster design manages to avoid contention in almost all cases. 

On the other hand, the swap system is complex. To show just how complex, Song put up this slide: 

> [ ![\[Swap system
diagram\]](https://static.lwn.net/images/conf/2025/lsfmm/KairuiSong-slide1.jpg)](<https://lwn.net/Articles/1016242/#slide>)

There are a lot of components with complex interactions, he said. New features are hard to add to this subsystem; the long (and ongoing) effort to add swapping for multi-size transparent huge pages (mTHPs) is one case in point. The whole thing is built around the one-byte-per-page design for the swap map, meaning that there is no space to store anything beyond a reference count and a pair of flags. Various optimizations have been bolted on over the years, increasing the complexity of the system. 

[![\[Kairui Song\]](https://static.lwn.net/images/conf/2025/lsfmm/KairuiSong-sm.png)](<https://lwn.net/Articles/1016242/>) As an example, he mentioned the `SWP_SYNCHRONOUS_IO` optimization, which was ~~[added to 6.14](<https://git.kernel.org/linus/1dd44c0af4fa>) by Baolin Wang~~ [added in 4.15](<https://git.kernel.org/linus/0bcac06f27d>) by Minchan Kim. When the kernel is swapping from a fast, memory-based device like [zram](<https://docs.kernel.org/admin-guide/blockdev/zram.html>), any extra latency hurts; in such cases, the kernel can simply bypass most of the swapping machinery and copy the data directly, preserving larger folios as well. Song said that this is a nice optimization, but there are now four different ways to bring in a folio from swap. He has tried to unify them all, but that work failed due to performance regressions. 

The distinction between the swap map and the swap cache adds complexity as well. The swap map is the one-byte-per-slot array that tracks the usage of swap slots in a swap device. It is a global resource, requiring locking for access, so it can be a contention point on systems that are doing a lot of swapping. The swap cache, instead, is a ~~per-CPU~~ data structure that holds a set of swap slots allocated in bulk from the swap map; it allows many swap-related actions to be done locklessly, and enables batching of swap-map changes when a lock must be taken. When a swap-map entry is in some CPU's cache, the `SWAP_HAS_CACHE` bit is set in the swap map to indicate that some CPU owns the entry. But, Song said, that bit has acquired other meanings over time, again making it harder to make changes to the swap machinery. 

Any redesign of this system, he said, is destined to use more memory; the one-byte design of the swap map just does not allow for much flexibility. There is, however, memory that could be used for this purpose. The swap cache currently uses eight bytes per slot, and control groups can add another two (duplicating some data in the process), so the actual memory consumption for swapping can be eleven bytes per slot. If that memory were to be repurposed, the swap map could be transformed into a "swap table" with eight bytes per entry, which would be enough for everything that he has in mind. Swap entries would still be managed in clusters, he said, and the total memory use of the swap subsystem should drop as some of the existing complexity is removed. 

Song's proposed layout for this swap table (taken from [his proposal for the session](<https://lwn.net/ml/all/CAMgjq7BvQ0ZXvyLGp2YP96+i+6COCBBJCYmjXHGBnfisCAb8VA@mail.gmail.com/>)) looks like this: 

```
| -----------    0    ------------| - Empty slot
        | SWAP_COUNT |---- Shadow ----|XX1| - Swapped out
        | SWAP_COUNT |------ PFN -----|X10| - Cached
        |----------- Pointer ---------|100| - Reserved / Unused yet
```

There would be a table for each swap cluster, spreading out the accesses and mostly eliminating locking contention; that, in turn, should allow the elimination of the separate swap cache. The eight-bit `SWAP_COUNT` is the same reference count that is kept in the current swap map, but it no longer needs to dedicate a couple of bits to flags like `SWAP_HAS_CACHE`. This design, he says, resolves many of the problems with the current swapping subsystem, and performs better as well, yielding a 10-20% improvement in the all-important kernel-build benchmark. There is only one swap-in path, and it never bypasses the table. Memory usage is lower, he said, and this design allows for the removal of a lot of complexity from the swap subsystem. 

A participant asked how the new design could be faster in the absence of the bypassing optimization currently used for zram. The answer was that the unification of the swap map and the swap cache means that there is no need to check or maintain both, making the swap subsystem as a whole faster. 

Future steps, Song said, include the addition of a virtual swap device that has no storage associated with it. Instead, it contains only entries pointing to slots in other swap devices. This new layer of indirection is intended to facilitate the intact swapping of larger folios, which currently becomes difficult when the swap devices become fragmented. The virtual device could be much larger than the physical swap space, making it resistant to fragmentation. 

Time was running out, so Song concluded with an idea to consider further in the future: swap migration. The list of swap clusters already works as a sort of least-recently-used (LRU) list, he said, so it could be used as a way of detecting folios that have been swapped out for a long time so that they could be moved to cheaper storage. Perhaps compaction could be performed at the same time. There was no time for the discussion of this idea, though. 

#### Virtual swap devices

The concept of a virtual swap device returned in the next session, though, as the topic that Pham wanted to talk about. His original motivation, he said, was to separate the [zswap](<https://docs.kernel.org/admin-guide/mm/zswap.html>) compressed swap cache from the rest of the swap subsystem. There is heavy use of zswap at his employer (Meta), which is good, but the current design of the swap layer requires that there be a slot assigned in an on-disk swap device for every page that is stored in zswap. That disk space will never be used and is thus entirely wasted; he has seen hosts running with an entirely unused, 100GB swap file. In an environment where hosts can have terabytes of installed RAM, it just is not possible to attach (and waste) sufficiently large drives for swap files. 

[![\[Nhat Pham\]](https://static.lwn.net/images/conf/2025/lsfmm/NhatPham-sm.png)](<https://lwn.net/Articles/1016252/>) Once a swap area has been added to the system, its size is fixed; the only way to increase swap space is to add another swap area. That is a slow operation, though, that a heavily loaded production system cannot afford, so Meta has to provision a suitably sized swap file ahead of time for each host type. There have been ongoing problems with machines running out of memory just because the unused swap device is "full". Pham appeared to be of the opinion that this was not an optimal way to run things. 

The problem, he said, is the tight coupling between swapped pages and the backing store behind them. The page-table entry for a swapped page points to the physical location for its data. It is, he said, a design oriented toward the sort of two-tier swapping system that was common some time ago. Beyond capacity problems, this design leads to other challenges; if, for example, zswap rejects a page, its page-table entry has already been changed and recovery is difficult. 

Solving this problem, he said, requires decoupling the various swap backends. A page stored in zswap should not take space in the other backends — unless that has been dictated by policy, as can happen with write-through caching. The system needs to be able to support multi-tier swapping; that would also help with the addition of new features, such as discontiguous swap-space allocation for large folios, or swapping in folios at different sizes than they were at swap-out time. 

Thus, he is proposing the implementation of a virtualized swap subsystem, providing swap space that is independent of whichever backend any given page is stored to. Each swapped-out page is assigned a virtual slot; that is what is stored in the page-table entry. Virtual slots can then be resolved to a specific backing store as needed, where that backing store could be zswap, a disk drive, a cache like zram, or something else. Such a system would eliminate the wasted space problem and allow pages to be moved between backends without having to change all of the references to them. That would make it easy for zswap to write pages back to another device, for example; it would also make removing a swap device much easier than it is now. 

He has a working prototype now, he said, that adds two new swap maps. There is a forward map that turns a virtual swap slot into a swap descriptor describing the actual placement of a page; it uses the [XArray](<https://lwn.net/Articles/745073/>) data structure, so lookups are lockless. The reverse map turns a physical swap slot into a virtual slot; that is useful to support cluster readahead or the movement of pages between backends. The metadata for a swapped page is placed in a dynamically allocated swap descriptor that is stored in the forward map. 

The prototype is getting close to the point where he can post it, he said. It is a big change, though, and he is worried about how he will be able to land it. Johannes Weiner suggested that it could perhaps operate in parallel with the existing swap subsystem until the performance is shown to be at least as good. 

At the end, a participant asked whether this system would be able to swap in a single page from a larger folio that has been swapped out; Pham said that he has considered that use in the design. Matthew Wilcox asked whether the virtual swap space would be used sparsely or densely; Pham, like Song, said that a large, sparse virtual space would be better for fragmentation avoidance. 

Pham has posted [the slides from this session](<https://drive.google.com/file/d/1mn2kSczvEzwq7j55iKhVB3SP67Qy4KU2/view>). 

#### Large-folio swap-in

The final episode of the swapping trilogy began with Arif reminding the crowd of the advantages of using large folios. They allow for better translation lookaside buffer (TLB) usage, reduce the number of page faults the system must handle, shorten LRU lists, and allow page-table entries to be manipulated in batches. Large folios often do not survive their encounter with the swap subsystem, though; they end up being split into base pages. Arif was there to talk about how the swap subsystem might be improved to better handle larger folios. 

[![\[Usama Arif\]](https://static.lwn.net/images/conf/2025/lsfmm/UsamaArif-sm.png)](<https://lwn.net/Articles/1016253/>) He mentioned work done by Ryan Roberts around a year ago to enable swapping out mTHPs without splitting them. That helped to avoid the cost of splitting these folios and avoid the fragmentation of memory. Work has been done to store large folios to zswap, and to be able to bring in large folios from zram. Compression of large folios in zram (which yields better compression) is being worked on, but has not been merged yet. One problem with compressing large folios, though, is that swapping in a single base page from that folio becomes difficult — the entire folio must be decompressed to make that base page accessible. 

Arif's large-folio swap-in work builds on these previous efforts. Specifically, at swap-in time, it checks to see whether the swap count is one (meaning there is a single reference to the page) and whether the page is in zswap. If so, the swap cache is skipped entirely, and zswap will be checked to see if it holds a larger folio containing the page in question. If so, the folio will be swapped in one page at a time and assembled into a proper folio at the end. 

This patch speeds 1MB folio swap-in by 36%, but also slows kernel builds. It resulted in an overall increase in zswap activity, with a lot of thrashing and folios being swapped in and out repeatedly. Thus, he concluded, perhaps large folios are not good for all workloads; would the group be happy with a change that yielded such different results for different workloads? Some workloads benefit; Android, for example, swaps out background tasks entirely and does not see this performance regression. Perhaps a control knob could be provided to tell the system whether to swap in large folios from zswap, but most users never change these knobs and would not see the benefit. Perhaps this behavior could be switched off automatically if the refault rate is seen as being too high. 

Another alternative would be to combine large-folio swapping with large-folio compression; that might offset the regression with kernel builds, he said. But the inability to swap in base pages out of large folios could get in the way here. 

As the session ran down, he wondered if there was a need for large-folio swap-in at all. Perhaps the system should continue to swap in base pages and let the `khugepaged` thread reassemble larger folios afterward. Wilcox said that there is a need to gather more statistics to understand what is really going on here. At this point, the topic was swapped out for something entirely different.
