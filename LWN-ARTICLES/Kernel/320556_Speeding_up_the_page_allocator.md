---
title: Speeding up the page allocator
url: https://lwn.net/Articles/320556/
date: "February 25, 2009"
category: "Memory management-Page allocator"
author: "By Jonathan Corbet February 25, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 25, 2009

It is a rare kernel operation that does not involve the allocation and freeing of memory. Beyond all of the memory-management requirements that would normally come with a complex system, kernel code must be written with extremely tight stack limits in mind. As a result, variables which would be declared as automatic (stack) variables in user-space code require dynamic allocation in the kernel. So the efficiency of the memory management subsystem has a pronounced effect on the performance of the system as a whole. That is why the kernel currently has three slab-level allocators (the original slab allocator, SLOB, and [SLUB](<http://lwn.net/Articles/229984/>)), with another one ([SLQB](<http://lwn.net/Articles/311502/>)) waiting for the 2.6.30 merge window to open. Thus far, nobody has been able to create a single slab allocator which provides the best performance in all situations, and the stakes are high enough to make it worthwhile to keep trying. 

While many kernel memory allocations are done at the slab level (using `kmem_cache_alloc()` or `kmalloc()`), there is another layer of memory management below the slab allocators. In the end, all dynamic memory management comes down to the page allocator, which hands out memory in units of full pages. The page allocator must manage memory without allowing it to become overly fragmented; it also must deal with details like CPU and NUMA node affinity, DMA accessibility, and high memory. It also clearly needs to be fast; if it is slowing things down, there is little that the higher levels can do to make things better. So one might do well to be concerned when memory management hacker Mel Gorman [writes](<http://lwn.net/Articles/320279/>): 

The complexity of the page allocator has been increasing for some time and it has now reached the point where the SLUB allocator is doing strange tricks to avoid the page allocator. This is obviously bad as it may encourage other subsystems to try avoiding the page allocator as well. 

As might be expected, Mel has come up with a set of patches designed to speed up the page allocator and do away the the temptation to try to work around it. The result appears to be a significant cleaning-up of the code and a real improvement in performance; it also shows the kind of work which is necessary to keep this sort of vital subsystem in top shape. 

Mel's 20-part patch (linked with the quote, above) attacks the problem in a number of ways. Many of them are small tweaks; for example, the core page allocation function (`alloc_pages_node()`) includes the following test: 

```
if (unlikely(order >= MAX_ORDER))
    	return NULL;
```

But, as Mel puts it, no proper user of the page allocator should be allocating something larger than `MAX_ORDER` in any case. So his patch set removes this test from the fast path of the allocator, replacing it with a rather more attention-getting test (`VM_BUG_ON`) in the slow path. The fast allocation path gets a little faster, and misuse of the interface should eventually be caught (and complained about) anyway. 

Then, there is the little function `gfp_zone()`, which takes the flags passed to the allocation request and decides which memory zone to try to allocate from. Different requests must be satisfied from different regions of memory, depending on factors like whether the memory will be used for DMA, whether high memory is acceptable, or whether the memory can be relocated if needed for defragmentation purposes. The current code accomplishes this test with a series of four `if` tests, but lots of jumps can be expensive in fast-path code. So Mel's patch replaces the tests with a table lookup. 

There are a number of other changes along these lines - seeming micro-optimizations that one would not normally bother with. But, in fast-path code deep within the system, this level of optimization can be worth doing. The patch set also reorganizes things to make the fast path more explicit and contiguous; that, too, can speed things up, but it also helps ensure that developers know when they are working with performance-critical code. 

The change which provoked the most discussion, though, was the removal of the distinction between hot and cold pages. This feature, [merged for 2.5.45](<http://lwn.net/Articles/14768/>), attempts to track which pages are most likely to be present in the processor's caches. If the memory allocator can give cache-warm pages to requesters, memory performance should improve. But, notes Mel, it turns out that very few pages are being freed as "cold," and that, in general, the decisions on whether to tag specific pages as being hot or cold are questionable. This feature adds some complexity to the page allocator and doesn't seem to improve performance, so Mel decided to take it out. After [running some benchmarks](<https://lwn.net/Articles/320568/>), though, he concluded that, in fact, he has no idea whether the feature helps or not. So the second version of the patch has left out the hot/cold removal, but this topic will be revisited in the future. 

Mel claims some good results: 

Running all of these through a profiler shows me the cost of page allocation and freeing is reduced by a nice amount without drastically altering how the allocator actually works. Excluding the cost of zeroing pages, the cost of allocation is reduced by 25% and the cost of freeing by 12%. Again excluding zeroing a page, much of the remaining cost is due to counters, debugging checks and interrupt disabling. Of course when a page has to be zeroed, the dominant cost of a page allocation is zeroing it. 

A number of standard user-space benchmarks also show improvements with this patch set. The reviews are generally good, so the chances are that these changes could avoid the lengthy delays that characterize memory management patches and head for the mainline in the relatively near future. Then there should be no excuse for trying to avoid the page allocator.
