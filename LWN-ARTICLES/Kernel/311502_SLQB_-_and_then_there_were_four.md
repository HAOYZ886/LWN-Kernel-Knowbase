---
title: "SLQB - and then there were four"
url: https://lwn.net/Articles/311502/
date: "December 16, 2008"
category: "Memory management-Slab allocators; Slab allocator"
author: "By Jonathan Corbet December 16, 2008"
---

By **Jonathan Corbet**  
December 16, 2008

The Linux kernel does not lack for low-level memory managers. The venerable slab allocator has been the engine behind functions like `kmalloc()` and `kmem_cache_alloc()` for many years. More recently, SLOB was added as a pared-down allocator suitable for systems which do not have a whole lot of memory to manage in the first place. Even more recently, [SLUB](<http://lwn.net/Articles/229984/>) went in as a proposed replacement for slab which, while being designed with very large systems in mind, was meant to be applicable to smaller systems as well. The consensus for the last year or so has been that at least one of these allocators is surplus to requirements and should go. Typically, slab is seen as the odd allocator out, but nagging doubts about SLUB (and some performance regressions in specific situations) have kept slab in the game. 

Given this situation, one would not necessarily think that the kernel needs yet another allocator. But Nick Piggin thinks that, despite the surfeit of low-level memory managers, there is always room for one more. To that end, he has developed the [SLQB allocator](<http://lwn.net/Articles/311093/>) which he hopes to eventually see merged into the mainline. According to Nick: 

I've kept working on SLQB slab allocator because I don't agree with the design choices in SLUB, and I'm worried about the push to make it the one true allocator. 

Like the other slab-like allocators, SLQB sits on top of the page allocator and provides for allocation of fixed-sized objects. It has been designed with an eye toward scalability on high-end systems; it also makes a real effort to avoid the allocation of compound pages whenever possible. Avoidance of higher-order (compound page) allocations can improve reliability significantly when memory gets tight. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

While there is a fair amount of tricky code in SLQB, the core algorithms are not that hard to understand. Like the other slab-like allocators, it implements the abstraction of a "slab cache" - a lookaside cache from which memory objects of a fixed size can be allocated. Slab caches are used directly when memory is allocated with `kmem_cache_alloc()`, or indirectly through functions like `kmalloc()`. In SLQB, a slab cache is represented by a data structure which looks very approximately like the following: 

![\[SLQB slab data structure\]](https://static.lwn.net/images/ns/kernel/slqb.png)

(Note that, to simplify the diagram, a number of things have been glossed over). 

The main `kmem_cache` structure contains the expected global parameters - the size of the objects being allocated, the order of page allocations, the name of the cache, etc. But scalability means separating processors from each other, so the bulk of the `kmem_cache` data structure is stored in per-CPU form. In particular, there is one `kmem_cache_cpu` structure for each processor on the system. 

Within that per-CPU structure one will find a number of lists of objects. One of those (`freelist`) contains a list of available objects; when a request is made to allocate an object, the free list will be consulted first. When objects are freed, they are returned to this list. Since this list is part of a per-CPU data structure, objects normally remain on the same processor, minimizing cache line bouncing. More importantly, the allocation decisions are all done per-CPU, with no bad cache behavior and no locking required beyond the disabling of interrupts. The free list is managed as a stack, so allocation requests will return the most recently freed objects; again, this approach is taken in an attempt to optimize memory cache behavior. 

SLQB gets its memory in the form of full pages from the page allocator. When an allocation request is made and the free list is empty, SLQB will allocate a new page and return an object from that page. The remaining space on the page is organized into a per-page free list (assuming the objects are small enough to pack more than one onto a page, of course), and the page is added to the `partial` list. The other objects on the page will be handed out in response to allocation requests, but only when the free list is empty. When the final object on a page is allocated, SLQB will forget about the page - temporarily, at least. 

Objects are, when freed, added to `freelist`. It is easy to foresee that this list could grow to be quite large after a burst of system activity. Allowing `freelist` to grow without bound would risk tying up a lot of system memory doing nothing while it is possibly needed elsewhere. So, once the size of the free list passes a watermark (or when the page allocator starts asking for help freeing memory), objects in the free list will be flushed back to their containing pages. Any partial pages which are completely filled with freed objects will then be returned back to the page allocator for use elsewhere. 

There is an interesting situation which arises here, though: remember that SLQB is fundamentally a per-CPU allocator. But there is nothing that requires objects to be freed on the same CPU which allocated them. Indeed, for suitably long-lived objects on a system with many processors, it becomes probable that objects will be freed on a different CPU. That processor does not know anything about the partial pages those objects were allocated from, and, thus, cannot free them. So a different approach has to be taken. 

That approach involves the maintenance of two more object lists, called `rlist` and `remote_free`. When the allocator tries to flush a "remote" object (one allocated on a different CPU) from its local `freelist`, it will simply move that object over to `rlist`. Occasionally, the allocator will reach across CPUs to take the objects from its local `rlist` and put them on `remote_free` list of the CPU which initially allocated those objects. That CPU can then choose to reuse the objects or free them back to their containing pages. 

The cross-CPU list operation clearly requires locking, so a spinlock protects `remote_free`. Working with the `remote_free` lists too often would thus risk cache line bouncing and lock contention, both of which are not helpful when scalability is a goal. That is why processors accumulate a group of objects in their local `rlist` before adding the entire list, in a single operation, to the appropriate `remote_free` list. On top of that, the allocator does not often check for objects in its local `remote_free` list. Instead, objects are allowed to accumulate there until a watermark is exceeded, at which point whichever processor added the final objects will set the `remote_free_check` flag. The processor owning the `remote_free` list will only check that list when this flag is set, with the result that the management of the `remote_free` list can be done with little in the way of lock or cache line contention. 

The SLQB code is relatively new, and is likely to need a considerable amount of work before it may find its way into the mainline. Nick claims benchmark results which are roughly comparable with those obtained using the other allocators. But "roughly comparable" will not, by itself, be enough to motivate the addition of yet another memory allocator. So pushing SLQB beyond comparable and toward "clearly better" is likely to be Nick's next task.
