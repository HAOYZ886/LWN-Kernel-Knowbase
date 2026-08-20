---
title: Toward a more efficient slab allocator
url: https://lwn.net/Articles/629152/
date: "January 13, 2015"
category: "Memory management-Slab allocators; Slab allocator"
author: "By Jonathan Corbet January 13, 2015 LCA 2015"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 13, 2015

* * *

[LCA 2015](<https://lwn.net/Archives/ConferenceByYear/#2015-linux.conf.au>)

Following up on [Jesper Brouer's session on networking performance](<https://lwn.net/Articles/629155/>), Christoph Lameter's LCA kernel miniconf session covered ways in which the performance of the kernel's low-level object allocators (the "slab" allocators) could be improved to meet current and future demands. Some of the work he covered is new, but some of it has been around, in concept at least, for some time. 

#### Batch allocation

Jesper talked about the need to process packets in batches; that, in turn, leads to the need to allocate and free data structures in batches. The overhead of a single-object allocation is too high for the needs of the networking subsystem, but, if that overhead can be spread out over a large numbers of objects, it becomes more tolerable. Christoph's work, which was [posted](<https://lwn.net/Articles/629356/>) to the linux-kernel list for review in December, provides an interface for multiple-object allocation. 

To allocate a set of objects from a slab cache, one would call: 

```
kmem_cache_alloc_array(struct kmem_cache *cache, gfp_t gfp, int nr,
        			   void **objects, unsigned int flags);
```

If all goes well, this function will allocate `nr` objects, placing pointers to them in the `objects` array. 

The `flags` argument is there to support a few different modes of allocation. `SLAB_ARRAY_ALLOC_LOCAL` says to allocate the objects from a local, per-CPU array. Allocation is lockless and quite fast, but there is a limited number of objects available from this cache. Larger batches can be allocated with `SLAB_ARRAY_ALLOC_PARTIAL`, which tries to grab the objects from the per-CPU list of partially-allocated pages. This mode may be a bit slower, but it avoids draining the per-CPU object cache. Finally, large numbers of objects can be allocated with `SLAB_ARRAY_ALLOC_NEW`, which allocates objects from freshly allocated pages. 

That last mode may seem especially slow since it requires calls into the page allocator. But, for large batches, Christoph said, it could actually be the fastest mode of them all. Normally the SLUB allocator (which Christoph maintains) must manipulate the free list of objects used in the management of slab pages; working with fresh pages avoids that need, and, in the process, cuts out a lot of cache misses associated with list traversal. The cost, in the current implementation, is that only full pages of objects can be allocated, so the returned number of objects may be less than what was asked for. Dave Chinner said that such an interface may be be useful in the filesystem layer, but the allocator would have to return the requested number of objects, so that behavior might change in the future. 

Objects can also be freed in batches, using: 

```
kmem_cache_free_array(struct kmem_cache *cache, int nr, void **objects);
```

The current plan is to add this array-allocation API with a fallback mode for slab allocators that do not support it. That allows testing the API without the need to implement it in all three allocators supported by the kernel. 

Implementation of this API in the SLUB allocator is done. The biggest challenge is the manipulation of the free lists, which can add a lot of cache misses to an allocation operation. As mentioned above, allocation using fresh pages avoids that problem, since the free list need never exist in the first place. Implementation in the slab allocator is easier, since it already maintains a per-page array of free objects; there is no free list to traverse. There was no mention of the SLOB allocator, but SLOB users are not primarily focused on performance anyway. 

#### Fixing slab fragmentation

The second part of Christoph's talk had to do with slab page fragmentation issues. All of the slab allocators work by allocating full pages, breaking them up into equal-sized objects, then passing those objects out to the the rest of the kernel on request. One result of this strategy is that, over time, the allocators accumulate lists of partially allocated pages — pages with [![\[Christoph Lameter\]](https://static.lwn.net/images/conf/2015/lca/ChristophLameter-sm.jpg)](<https://lwn.net/Articles/629277/>) some objects allocated, and others free. These fragmented pages are costly to track; they also represent a fair amount of wasted memory that cannot be freed for other uses. There would be value in a mechanism that could free some of these partially allocated pages. 

There are a number of patches out there addressing parts of the fragmentation problem. The first of these takes a relatively simple approach: the lists of partially allocated pages are sorted to put those with the fewest free objects at the beginning. The hope is that subsequent allocation requests will allocate the last remaining objects in those pages, at which point the allocator can stop tracking them. At the other end of the list, the pages which contain few allocated objects will, with luck and if further objects are not allocated from them, become fully free when the remaining objects are returned. Those pages can then be handed back to the page allocator. 

The next step is off-node allocation. The slab allocators normally try to keep memory allocations on the same NUMA node as the requester. But, on occasion, the SLUB allocator will allocate from a remote node in the hope of clearing some partially-allocated pages from that node. This off-node access happens relatively rarely, and only if the allocation request does not explicitly ask for node-local memory. But, carefully used, it can help to get mostly-allocated pages off the partial-page lists. 

A more invasive approach is what Christoph called "defragmentation by eviction." It was first posed in 2009, but was rejected at the time. It allows callbacks to be associated with objects allocated from a slab cache. There are two of these: `get()` and `kick()`. A call to `get()` establishes a stable reference to an object so that it will not be freed while the allocator is trying to free the entire page. A call to `kick()`, instead, requests that the object be freed. The callback can refuse to free the object, but, clearly, the mechanism will work better if these requests are honored whenever possible. After all, it only takes one refused request to thwart an attempt to free a page. 

Finally, Christoph mentioned that, sometime in the future, there will be a need to support movable objects in the slab caches. Much work has gone into making memory pages movable; at this point, the slab caches represent the bulk of unmovable pages in the system. Solving that problem will not be easy, Christoph said, but it may, in the end, be the only way to truly solve the problem of slab page fragmentation. 

[Your editor would like to thank linux.conf.au for funding his travel to the event.]
