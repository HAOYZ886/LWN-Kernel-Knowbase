---
title: The SLUB allocator
url: https://lwn.net/Articles/229984/
date: "April 11, 2007"
category: "Memory management-Slab allocators; Slab allocator"
author: 
---

The slab allocator has been at the core of the kernel's memory management for many years. This allocator (sitting on top of the low-level page allocator) manages caches of objects of a specific size, allowing for fast and space-efficient allocations. Kernel hackers tend not to wander into the slab code because it's complex and because, for the most part, it works quite well. 

Christoph Lameter is one of those people for whom the slab allocator does not work quite so well. Over time, he has come up with a list of complaints that is getting impressively long. The slab allocator maintains a number of queues of objects; these queues can make allocation fast but they also add quite a bit of complexity. Beyond that, the storage overhead tends to grow with the size of the system: 

SLAB Object queues exist per node, per CPU. The alien cache queue even has a queue array that contain a queue for each processor on each node. For very large systems the number of queues and the number of objects that may be caught in those queues grows exponentially. On our systems with 1k nodes / processors we have several gigabytes just tied up for storing references to objects for those queues This does not include the objects that could be on those queues. One fears that the whole memory of the machine could one day be consumed by those queues. 

Beyond that, each slab (a group of one or more continuous pages from which objects are allocated) contains a chunk of metadata at the beginning which makes alignment of objects harder. The code for cleaning up caches when memory gets tight adds another level of complexity. And so on. 

Christoph's response is the [SLUB allocator](<http://lwn.net/Articles/229096/>), a drop-in replacement for the slab code. SLUB promises better performance and scalability by dropping most of the queues and related overhead and simplifying the slab structure in general, while retaining the current slab allocator interface. 

In the SLUB allocator, a slab is simply a group of one or more pages neatly packed with objects of a given size. There is no metadata within the slab itself, with the exception that free objects are formed into a simple linked list. When an allocation request is made, the first free object is located, removed from the list, and returned to the caller. 

Given the lack of per-slab metadata, one might well wonder just how that first free object is found. The answer is that the SLUB allocator stuffs the relevant information into the system memory map - the `page` structures associated with the pages which make up the slab. Making `struct page` larger is frowned upon in a big way, so the SLUB allocator makes this complicated structure even more so with the addition of another union. The end result is that `struct page` gets three new fields which only have meaning when the associated page is part of a slab: 

```
void *freelist;
        short unsigned int inuse;
        short unsigned int offset;
```

For slab use, `freelist` points to the first free object within a slab, `inuse` is the number of objects which have been allocated from the slab, and `offset` tells the allocator where to find the pointer to the next free object. The SLUB allocator can use RCU to free objects, but, to do so, it must be able to put the "next object" pointer outside of the object itself; the `offset` pointer is the allocator's way of tracking where that pointer was put. 

When a slab is first created by the allocator, it has no objects allocated from it. Once an object has been allocated, it becomes a "partial" slab which is stored on a list in the `kmem_cache` structure. Since this is a patch aimed at scalability, there is, in fact, one "partial" list for each NUMA node on the system. The allocator tries to keep allocations node-local, but it will reach across nodes before filling the system with partial slabs. 

There is also a per-CPU array of active slabs, intended to prevent cache line bouncing even within a NUMA node. There is a special thread which runs (via a workqueue) which monitors the usage of per-CPU slabs; if a per-CPU slab is not being used, it gets put back onto the partial list for use by other processors. 

If all objects within a slab are allocated, the allocator simply forgets about the slab altogether. Once an object in a full slab is freed, the allocator can relocate the containing slab via the system memory map and put it back onto the appropriate partial list. If all of the objects within a given slab (as tracked by the `inuse` counter) are freed, the entire slab is given back to the page allocator for reuse. 

One interesting feature of the SLUB allocator is that it can combine slabs with similar object sizes and parameters. The result is fewer slab caches in the system (a 50% reduction is claimed), better locality of slab allocations, and less fragmentation of slab memory. The patch does note: 

Note that merging can expose heretofore unknown bugs in the kernel because corrupted objects may now be placed differently and corrupt differing neighboring objects. Enable sanity checks to find those. 

Causing bugs to stand out is generally considered to be a good thing, but wider use of the SLUB allocator could lead to some quirky behavior until those new bugs are stamped out. 

Wider use may be in the cards: the SLUB allocator is in the -mm tree now and could hit the mainline as soon as 2.6.22. The simplified code is attractive, as is the claimed 5-10% performance increase. If merged, SLUB is likely to coexist with the current slab allocator (and the SLOB allocator intended for small systems) for some time. In the longer term, the current slab code may be approaching the end of its life.
