---
title: "A CLOCK-Pro page replacement implementation"
url: https://lwn.net/Articles/147879/
date: "August 16, 2005"
category: "CLOCK-Pro; Memory management-Page replacement algorithms"
author: 
---

Virtual memory management appears to be a perennially unsolved operating systems problem. Nobody has yet figured out how to perform page replacement in such a way as to ensure that the pages that will be needed in the future may be found in main memory. Crystal balls, it seems, remain fiendishly difficult to implement. 

The reigning algorithm used in most systems is a variant of the least-recently-used (LRU) scheme. If a page has not been used in a long time, the reasoning goes, it probably will not be needed again in the near future; pages which have not been used for a while are thus candidates for eviction from main memory. In practice, tracking the usage of every page would impose an unacceptable amount of overhead, and is not done. Instead, the VM subsystem scans sequentially through the "active list" of pages in use, marking them as "inactive." Pages on the inactive list are candidates for eviction. Some of those pages will certainly be needed soon, however, with the result that they will be referenced before that eviction takes place. When this happens, the affected pages are put back on the active list at the "recently used" end. As long as pages stay in the inactive list for a reasonable time before eviction, this algorithm approximates a true LRU scheme. 

This mechanism tends to fall apart with certain types of workloads, however. Actions like initializing a huge array, reading a large file (for streaming media playback, for example), starting OpenOffice, or walking through a large part of the filesystem can fill main memory with pages which are unlikely to be used again anytime soon - at the expense of the pages the system actually needs. Pages from files start in the inactive list and may, at least, be shoved back out relatively quickly, but anonymous memory pages go straight to the active list. Many Linux users are familiar with the occasional sluggish response which can come after the active list has been flushed in this way; with some workloads, this behavior can be a constant thing, and the system will consistently perform poorly. 

Rik van Riel has recently posted a set of patches aimed at improving the performance of the VM subsystem under contemporary loads. The algorithm implemented is based on [CLOCK-Pro](<http://www.cs.wm.edu/hpcs/WWW/HTML/publications/abs05-3.html>), developed by Song Jiang, Feng Chen, and Xiaodong Zhang. CLOCK-Pro attempts to move beyond the LRU approach by tracking how often pages are accessed and tweaking the behavior of the VM code to match. At its core, CLOCK-Pro tries to ensure that pages in the inactive list are referenced less frequently than those on the active list. It thus differs from LRU ![\[access diagram\]](https://static.lwn.net/images/ns/kernel/clock-pro-accesses.png) schemes, which prioritize the most recently accessed pages even if those particular pages are almost never used by the application. Consider, as an example, the diagram to the right showing access patterns for two pages. At the time `t1` marked by the red line, an LRU algorithm would prefer page 2 over page 1, even though the latter is more likely to be used again in the near future. 

Implementing CLOCK-Pro requires that the kernel keep track of pages which have recently been evicted from main memory. To this end, Rik's patches create a new data structure which tries to perform this tracking without adding much extra overhead. There is a new kernel function: 

```
int do_remember_page(struct address_space *mapping, unsigned long index);
```

The VM code will, when moving a page out of main memory, first call `remember_page()` with the relevant information. This function implements a data structure which looks a little like the following: 

![\[Cheezy nonresident page diagram\]](https://static.lwn.net/images/ns/kernel/nonresident_page.png)

When a page is to be remembered, a hash value is generated from the `mapping` and `index` parameters; this value will be used as an index into the `nonres_table` array. Each hash bucket contains a fixed number of entries for nonresident pages. `do_remember_page()` treats the hash bucket like a circular buffer; it will use the `hand` index to store a cookie representing the page (a separate hash, essentially) in the next available slot, possibly overwriting information which was there before. The size of the entire data structure is chosen so that it can remember approximately as many evicted pages as there are pages of real memory in the system. The cost of the structure is one 32-bit word for each remembered page. 

At some point in the future, the kernel will find itself faulting a page into memory. It can then see if it has seen that page before with a call to: 

```
int recently_evicted(struct address_space *mapping, unsigned long index);
```

A non-negative return value indicates that the given page was found in the nonresident page cache, and had, indeed, been evicted not all that long ago. The return value is actually an estimate of the page's "distance" - a value which is taken by seeing how far the page's entry is from the current value of the `hand` index (in a circular buffer sense) and scaling it by the size of the array. In a rough sense, the distance is the number of pages which have been evicted since the page of interest was pushed out. 

Whenever a page is faulted in, the kernel computes a distance for the oldest page in the active list; this distance is an estimate taken from how long ago the oldest page would have been scanned (at the current rate). This distance is compared to the distance of the newly-faulted page (which is scaled relative to the total number of recently evicted pages) to get a sense for whether this page (which had been evicted) has been accessed more frequently than the oldest in-memory page. If so, the kernel concludes that the wrong pages are in memory; in response, it will decrease the maximum desired size of the active list to make room for the more-frequently accessed pages which are languishing in secondary storage. The kernel will also, in this case, add the just-faulted page directly to the active list, on the theory that it will be useful for a while. 

If, instead, pages being faulted in are more "distant" than in-core pages, the VM subsystem concludes that it is doing the right thing. In this situation, the size of the active list will be slowly increased (up to a maximum limit). More distant pages are faulted in to the inactive list, meaning that they are more likely to be evicted again in the near future. 

Your editor applied the patch to a vanilla 2.6.12 kernel and ran some highly scientific tests: a highly parallel kernel make while simultaneously running a large "`grep -r` to read large amounts of file data into the page cache. The patched kernel adds a file (`/proc/refaults`) which summarizes the results from the nonresident page cache; after this experiment it looked like this: 

```
Refault distance          Hits
            0 -      4096           138
         4096 -      8192           108
         8192 -     12288            93
        12288 -     16384            88
        16384 -     20480            86
        20480 -     24576            84
        24576 -     28672            59
        28672 -     32768            48
        32768 -     36864            53
        36864 -     40960            46
        40960 -     45056            43
        45056 -     49152            46
        49152 -     53248            39
        53248 -     57344            39
        57344 -     61440            39
     New/Beyond     61440         11227
```

This histogram shows that the vast majority of pages brought into the system had never been seen before; they would be mainly the result of the large `grep`. A much smaller number of pages - a few hundred - had very small distances. If the patch is working right, those pages (being, one hopes, important things like the C compiler) would be fast-tracked into the active list while the large number of unknown pages would be hustled back out of main memory relatively quickly. 

As it turns out, [the patch doesn't work right](<https://lwn.net/Articles/147898/>) quite yet. Much of the structure is in place, but the desired results are not yet being seen. These details will presumably be worked out before too long. Only at that point will it be possible to benchmark the new paging code and decide whether it truly performs better or not. One never knows ahead of time with virtual memory code; the proof, as they say, is in the paging. 

[Thanks to Rik van Riel for his review of a previous draft of this article.]
