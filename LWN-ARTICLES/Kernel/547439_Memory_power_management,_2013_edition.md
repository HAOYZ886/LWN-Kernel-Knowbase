---
title: "Memory power management, 2013 edition"
url: https://lwn.net/Articles/547439/
date: "April 17, 2013"
category: "Memory management-Power management; Power management"
author: "By Jonathan Corbet April 17, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 17, 2013

When developers talk about power management, they are almost always concerned with the behavior of the CPU, since that is where the largest savings tend to be found. Computers are made up of more than just CPUs, though, and the other components require power as well. Seemingly, about once per year, attention turns to reducing the power demands of the RAM on the system; since RAM can take up to one third of a system's total power budget, this focus makes sense. Accordingly, LWN has looked at this issue [once in 2011](<https://lwn.net/Articles/446493/>) and [again in 2012](<https://lwn.net/Articles/479886/>). Now there is a new memory power management patch set in circulation, so another look seems warranted. 

The [most recent patch set](<https://lwn.net/Articles/546696/>) comes from Srivatsa S. Bhat; it differs from previous approaches in a number of ways. For example, it targets memory controllers that have automatic, content-preserving power management modes. Such controllers divide memory into a set of regions, each of which can be powered down independently when the controller detects that there have been no memory accesses to the region in the recent past. The strategy to use is fairly obvious: try to keep as many memory regions as possible empty so that they will stay powered down. 

The first step is to keep track of those regions in the memory management subsystem. Previous patches have used the zone system (which divides memory with different characteristics -- high and low memory on 32-bit systems, for example) to track regions. The problem with this approach is that it causes an explosion in the number of zones; that leads to more memory management overhead and challenges in keeping memory usage balanced across all those zones. Srivatsa's patch, instead, tracks regions as separate entities in parallel with zones, avoiding this problem. 

Once the kernel knows where the regions are, the trick is to concentrate memory allocations on a relatively small number of those regions whenever possible. To that end, the patch set causes the list of free pages to be sorted by region, so that allocations from the head of the list will come from the lowest-numbered region with available pages. Note that sorting _within_ a region is not necessary; it is sufficient that all pages in a given region are grouped together. A set of pointers into the free list, one per region, helps newly-freed pages to be quickly added to the list in the correct location. 

Region-aware allocation can help to keep active pages grouped together, but, in the real world, allocated pages will still end up being spread across physical memory over time. Unless other measures are taken, most regions will end up with active pages even when the system is under relatively light memory load; that will make powering down those regions difficult or impossible. So, inevitably, Srivatsa's patch set includes a mechanism for migrating pages out of regions. 

Vacating regions of memory is not a new problem; the [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>) mechanism must sometimes take active measures to create large contiguous blocks, for example. So this particular problem has already been solved in the kernel. Rather than add a new compaction scheme, Srivatsa's patch set modifies the CMA implementation to make it suitable for memory power management uses as well. The result is a simple `compact_range()` function that can be invoked by either subsystem to move pages and free a range of memory. 

There is still the question of _when_ the kernel should try to vacate a memory region. If it does not happen often enough, power consumption will be higher than it needs to be. Excessive page migration will simply soak up CPU time, though, with no resulting power savings. Indeed, overly aggressive compaction could result in higher power usage than before. So some sort of control mechanism is required. 

In this patch set, the page allocator has been enhanced to notice when it starts allocating pages from a new memory region. That new region, by virtue of having been protected from allocations until now, should not have many pages allocated; that makes it a natural target for compaction. But it makes no sense to attempt that compaction when the page is being allocated, since, clearly, no free pages exist in the lower-numbered regions. So the page allocator does not attempt compaction at that time; instead, it sets a flag indicating that compaction should be attempted in the near future. 

The "near future" means when pages are freed. When some pages are given back to the allocator, it might be possible to use those pages to free a lightly-used region of memory. So that is the time when compaction is attempted; a workqueue function will be kicked off to attempt to vacate any regions that had previously been marked by the allocator. That code will only make the attempt, though, if a relatively small number of pages (32 in the current patch) would need to be migrated. Otherwise the cost is deemed to be too high and the region is left alone. 

The patch set is still young, so there is not a lot of performance data available. In the introduction, though, a 6% power savings is claimed when running on a 2GB Samsung Exynos board, with the potential for more held out if other parts of the memory management subsystem can be made more power aware. One question that is not answered in the patch set is this: on a typical Linux system, very few pages are truly "free"; instead, they are occupied by the page cache. To be able to vacate regions, it seems like a more aggressive approach to reclaiming page-cache pages would be required. There are undoubtedly other concerns that would need to be addressed as well; perhaps they will be discussed in the 2014 update, if not before.
