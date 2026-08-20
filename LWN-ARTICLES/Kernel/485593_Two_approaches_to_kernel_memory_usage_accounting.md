---
title: Two approaches to kernel memory usage accounting
url: https://lwn.net/Articles/485593/
date: "March 7, 2012"
category: "Control groups-Memory controller; Memory management-Control groups"
author: "By Jonathan Corbet March 7, 2012"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 7, 2012

The kernel's memory usage controller allows a system administrator to place limits on the amount of memory used by a given control group. It is a useful tool for systems where memory usage policies must be applied - often systems where virtualization or containers are being used - but it has one notable shortcoming: it only tracks user-space memory. The memory used by the kernel on behalf of a control group is not tracked. For some workloads, the amount of memory involved may be considerable; a control group that accesses large numbers of files, for example, will create a lot of entries in the kernel's directory entry ("dentry") cache. Without the ability to control this kind of memory use in the kernel, the memory controller remains a partial solution. 

Given that, it should not be surprising that a patch set adding the ability to track and limit kernel memory use exists. What may be a little more surprising is the fact that two independent patch sets exist, each of which adds that feature in its own way. Both were posted for consideration in late February. 

The first set was [posted](<https://lwn.net/Articles/482777/>) by Glauber Costa, the author of the related [per-cgroup TCP buffer limits](<https://lwn.net/Articles/470656/>) controller. Glauber's patch works at the slab allocator level; only the SLUB allocator is supported at this time. With this approach, developers must explicitly mark a slab cache for usage tracking with this interface: 

```
struct memcg_cache_struct {
    	int index;
    	struct kmem_cache *cache;
    	int (*shrink_fn)(struct shrinker *shrink, struct shrink_control *sc);
    	struct shrinker shrink;
        };
    
        void register_memcg_cache(struct memcg_cache_struct *cache);
```

Once a slab cache has been passed to `register_memcg_cache()`, it is essentially split into an array of parallel caches, one belonging to each control group managed by the memory controller. With some added infrastructure, each of these per-cgroup slab caches is able to track how much memory has been allocated from it; this information can be used to cause allocations to fail should the control group's limits be exceeded. More usefully, the controller can, when limits are exceeded, call the `shrink_fn()` associated with the cache; that function's job is to find memory to free, bringing the control group back below its limit. 

Glauber's patch set includes a sample implementation for the dentry cache. When a control group creates enough dentries to run past its limits, the shrinker function can clean some of them up. That may slow down processes in the affected control group, but it should prevent a dentry-intensive process from affecting processes in other control groups. 

The [second patch set](<https://lwn.net/Articles/484056/>) comes from Suleiman Souhlal. Here, too, the slab allocator is the focus point for memory allocation tracking, but this patch works with the "slab" allocator instead of SLUB. One other significant difference with Suleiman's patch is that it tracks allocations from _all_ caches, rather than just those explicitly marked for such tracking. There is a new `__GFP_NOACCOUNT`. flag to explicitly prevent tracking, but, as a whole, it's an opt-out system rather than opt-in. One might argue that, if tracking kernel memory usage is important, one should track all of it. But, as Suleiman acknowledges, the ability to track allocations from all caches ""is also the main source of complexity in the patchset"". 

Under this scheme, slab caches operate as usual until an allocation is made from a specific cache while under the control of a specific cgroup. At that point, the cache is automatically split into per-cgroup caches without the intervention (or knowledge) of the caller. Of course, this splitting requires taking locks and allocating memory - activities that can have inconvenient results if the system is running in an atomic context at the time. In such situations, the splitting of the cache will be pushed off into a workqueue while the immediate allocation is satisfied from the pre-split cache. Much of the complexity in Suleiman's patch set comes from this magic splitting that works regardless of the calling context. 

There is no shrinker interface in this patch set, though that is clearly planned for the future. 

When a control group is deleted, both implementations shift the accounting up to the parent group. That operation, too, can involve some complexity; the processes that performed the allocation may, like their control group, be gone when the allocations are finally freed. Glauber's patch does no tracking for the root control group; as a result of that decision (and some careful programming), the cost of the kernel memory tracking feature is almost zero if it is not actually being used. Suleiman's patch does track usage for the root cgroup, but that behavior can be disabled with a kernel configuration option. 

Neither patch appears to be ready for merging into the mainline prior to the 3.5 development cycle - and, probably, not even then. There are a lot of details to be worked out, the mechanism needs to work with both slab and SLUB (at least), and, somehow, the two patch sets need to turn into a single solution. The two developers are talking to each other and express interest in working together, but there will almost certainly need to be guidance from others before the two patches can be combined. If users of this feature feel that tracking allocations from all slab caches is important, then, clearly, whatever is merged will need to have that feature. If, instead, picking a few large users is sufficient, then a solution requiring the explicit marking of caches to be tracked will do. Thus far, there has not been a whole lot of input from people other than the two developers; until that happens, it will be hard to know which approach will win out in the end.
