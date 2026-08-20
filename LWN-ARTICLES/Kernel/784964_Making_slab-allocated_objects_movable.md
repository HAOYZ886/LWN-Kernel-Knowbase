---
title: "Making slab-allocated objects movable"
url: https://lwn.net/Articles/784964/
date: "April 8, 2019"
category: "Memory management-Slab allocators"
author: "By Jonathan Corbet April 8, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 8, 2019

Memory fragmentation is a constant problem for memory-management subsystems. Over the years, considerable effort has been put into reducing fragmentation in the Linux kernel, but almost all of that work has been focused on memory management at the page level. The slab allocators, which (mostly) manage memory in chunks of less than the page size, have seen less attention, but fragmentation at this level can create problems throughout the system. The [slab movable objects patch set](<https://lwn.net/ml/linux-kernel/20190403042127.18755-1-tobin@kernel.org/>) posted by Tobin Harding is an attempt to improve this situation by making it possible for the kernel to actively defragment slab pages by moving objects around. 

Over the course of normal operation, the kernel allocates (and sometimes frees) vast numbers of small objects in memory. The slab allocators are designed to make these allocation operations efficient; they allocate whole pages, then parcel out the smaller objects from there. Sets of pages ("slabs") are set aside for objects of a fixed size, allowing them to be efficiently packed with a minimum of overhead and waste. Linux users can choose between three slab allocators: the original allocator (simply called "slab"), SLUB (the newer allocator used on most systems), and SLOB (a minimal allocator for the smallest systems). 

For a window into how the allocator on a given system is working, one can look at `/proc/slabinfo`. On your editor's system, for example, there are currently 338,860 active dentry cache entries, each of which requires an object from the slab allocator. A `dentry` structure is 192 bytes on this system, so 21 of them can be allocated from each full page. Thus, a minimum of 16,136 pages are needed to hold these objects; on the system in question, 16,461 are actually used for that purpose. There are thus over 300 pages allocated beyond what is strictly needed, which is actually a relatively low level of overhead; it can get a lot worse. 

When the system runs low on memory, it will call back to owners of various slabs to request that they free objects to make memory available for other use. The recipients of these calls will dutifully free some objects if they can, but this often is not as useful as one would like. The slab allocator can only return a page to the system if all of the objects on that page have been freed. If the dentry cache, for example, frees 100,000 of those 338,860 objects, it will have certainly made the cache hit rate lower, but since those objects may be scattered throughout those 16,461 pages, the number of pages actually freed might be quite small. That can be a significant performance hit for little memory gain, but that is how things work now. A small number of objects can pin a lot of pages that are mostly unused, wasting a lot of memory. 

It would be better if the slab allocator could move allocated objects out of slab pages that are mostly empty, freeing those pages while making better use of other pages with free space in them. The problem, of course, is that any such mechanism requires cooperation from whoever is allocating objects from the slab. The owners of those objects access them with direct pointers; if an object is to be moved, any pointers to it will have to be located and changed. That complicates the picture considerably and, for slabs that allocate objects for many callers (those that handle `kmalloc()`, for example), making objects movable is not really feasible. There are other cases, though, where a single owner exists; for those, getting the owner to move things might just be possible. That is the idea behind the slab movable objects patches, which adds object mobility to the SLUB allocator. 

To support movable objects, the owner of a slab cache must provide two new functions. The first, called `isolate()`, has this prototype: 

```
typedef void *kmem_cache_isolate_func(struct kmem_cache *s, void **objs, int nr);
```

A call to this function tells the owner that the slab cache would like to relocate `nr` objects in the cache `s`, the addresses of which are stored in `objs`. The objects should not actually be moved at this time, but they should be locked or otherwise frozen so that no other changes are made to them while the process is underway. If it is known that any of the objects cannot be moved, their pointer can be zeroed out in `objs`. Should it be necessary to retain any data about this relocation, the function can return a pointer to that data. 

The next step is to actually move the objects with the `migrate()` function: 

```
typedef void kmem_cache_migrate_func(struct kmem_cache *s, void **objs,
    				         int nr, int node, void *private);
```

The `s`, `objs`, and `nr` parameters are as with `isolate()`. The `node` argument indicates a NUMA node where the objects should be moved to, and `private` is the pointer that was returned from `isolate()`. The function should actually move the objects, typically by calling `kmem_cache_alloc()` to allocate new objects, copying the data over, and updating any internal pointers. The old objects should then be freed. Any locking that was applied to these objects in the `isolate()` function should, of course, be undone here. 

To enable object mobility for a given slab cache, the above functions should be passed to: 

```
void kmem_cache_setup_mobility(struct kmem_cache *s,
        				   kmem_cache_isolate_func isolate,
    			           kmem_cache_migrate_func migrate);
```

There is one other requirement for mobility to work: the slab cache must have a constructor associated with it. That is because the kernel might try to relocate objects at any time, and that can require dealing with the data in those objects. If they are not all properly initialized and consistent from the outset, bad things could happen. 

The patch set enables object relocation in two subsystems: the dentry cache and the XArray data structure. The dentry cache implementation is relatively simple; rather than try to relocate cache entries, it simply frees those that have been targeted. One might argue that the functionality is similar to how the cache shrinker works now, but there is a difference: the objects to be freed can be chosen to free up specific pages in the slab cache. It should, thus, be rather more efficient. That said, some problems with the current dentry cache implementation were [pointed out](<https://lwn.net/ml/linux-kernel/20190403170811.GR2217@ZenIV.linux.org.uk/>) by Al Viro; some work will need to be done there before this code is ready for the mainline. 

The XArray implementation is simpler; there is no need for an `isolate()` function, so none is provided. The `migrate()` function is able to take locks and reallocate objects relatively easily without any advance preparation. 

Making slab objects movable will not solve the problem of slab-cache fragmentation on its own. But, if applied to the largest caches in the system, it should be able to improve the situation considerably. That, in turn, will make progress on a problem that has affected the memory-management subsystem for years.
