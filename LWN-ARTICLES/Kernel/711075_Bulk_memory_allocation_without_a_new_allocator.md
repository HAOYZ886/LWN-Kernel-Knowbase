---
title: Bulk memory allocation without a new allocator
url: https://lwn.net/Articles/711075/
date: "January 10, 2017"
category: "Memory management-Scalability"
author: "By Jonathan Corbet January 10, 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 10, 2017

The kernel faces a number of scalability challenges resulting from the increasing data rates that can be handled by peripherals like storage devices and network interfaces. Often, the key to improved throughput is doing work in batches; in many cases, the overhead of performing a series of related operations is not much higher than for performing a single operation. Memory allocation is one place where batching offers the potential for significant performance improvements, but there has, so far, been no agreement on how that batching should be done. A new patch set from Mel Gorman might just show how this problem can be solved. 

Network interfaces tend to require a lot of memory; all those incoming packets have to be put somewhere, after all. But the overhead of allocating that memory is high, to the point that it can limit the maximum throughput of the system as a whole. In response, driver developers are resorting to workarounds like allocating (then splitting up) high-order pages, but high-order page allocation can stress the system as a whole and runs counter to normal kernel development practice. It would be good to have a better way. 

At the 2016 Linux Storage, Filesystem, and Memory-Management Summit, networking developer Jesper Dangaard Brouer [proposed](<https://lwn.net/Articles/684616/>) the creation of a new memory allocator designed from the beginning for batch operations. Drivers could use it to allocate many pages in a single call, thus minimizing the per-page overhead. The memory-management developers at this session understood the problem, but disagreed with the idea of creating a new allocator. Doing so, they said, would make the memory-management subsystem less maintainable. Additionally the new allocator would tend to repeat the mistakes of the existing allocators and, by the time it had all the necessary features, it might not be any faster. 

The right solution, from the memory-management perspective, is to modify the existing page allocator, reducing overheads and making it more friendly to multi-page allocations. This has not been done so far for a simple reason: most memory users immediately zero every page they allocate, an operation that is far more expensive than the allocation itself. That zeroing is not necessary for pages that will be overwritten with incoming packet data by a network interface, though, so high-performance networking workloads are more seriously affected by the overhead in the allocator. Fixing that overhead in the existing page allocator would solve the problem for the networking subsystem while avoiding the addition of a new allocator and providing improved performance for all parts of the kernel. 

The idea made sense, but only had one shortcoming: nobody had actually done the work to improve the existing page allocator in this way. That situation has changed, though, with the posting of Gorman's [bulk page allocator](<https://lwn.net/Articles/711046/>) patch set. The patches are relatively small, but the claimed result is a significant improvement in page-allocator performance. 

Two fundamental changes are required to support both allocations; both take the same form. The first of these addresses the function `buffered_rmqueue()`, which removes a page from a per-CPU free list in preparation for handing it out in response to an allocation request. Since the list is per-CPU, there is no locking required before making changes, but it is necessary to disable interrupts on the relevant CPU to prevent concurrent access from an interrupt handler. Disabling and restoring interrupts takes some significant time, and that time adds up if it must be done repeatedly for each page being allocated. 

Gorman's patch set splits up this function in a way that is common in kernel programming. A new function (`__rmqueue_pcplist()`) removes a page from the list but does not concern itself with disabling interrupts; that is expected to be handled by the caller. A call to `rmqueue_pcplist()` (without the leading underscores) will disable interrupts and allocate the page in the usual way. But now other code can disable interrupts once, then call `__rmqueue_pcplist()` multiple times to allocate a whole set of pages. 

Similarly, `__alloc_pages_nodemask()` spends a fair amount of time figuring out which zone of memory should be used to satisfy a given request, then returns a page. Here, too, those two operations can be split apart, so that the zone calculation can be reused for multiple page allocations rather than being performed anew for each page. 

With these two changes in place, Gorman's patch set can add a new allocation function: 

```
unsigned long alloc_pages_bulk(gfp_t gfp_mask, unsigned int order,
    				   unsigned long nr_pages, struct list_head *list);
```

This function will attempt to allocate `nr_pages` pages in an efficient manner, storing them in the given `list`. The `order` argument suggests that any size of allocation can be done in bulk but, in the current patch, any `order` other than zero (single pages) will result in a failure return. 

The result of using this interface, he says, is a ""roughly 50-60% reduction in the cost of allocating pages"". That should help the networking developers in their quest to improve packet throughput rates. They will find that some assembly is required, though; Gorman went as far as to show that the memory-allocator overhead can be reduced, but stopped short of creating an API with all of the features that those developers need. His [plan](<https://lwn.net/Articles/711164/>) is to merge the preparatory patches without the `alloc_pages_bulk()` API with the idea that the actual bulk-allocation API should be designed by the developers who need it. Thus, once these changes find their way into the mainline, it will be up to the networking crew to do something useful with them.
