---
title: Smarter shrinkers
url: https://lwn.net/Articles/550463/
date: "May 14, 2013"
category: "Memory management-Shrinkers"
author: "By Jonathan Corbet May 14, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 14, 2013

One of the kernel's core functions is the management of caching; by maintaining caches at various levels, the kernel is able to improve performance significantly. But caches cannot be allowed to grow without bound or they will eventually consume all of memory. The kernel's answer to this problem is the "shrinker" interface, a mechanism by which the memory management subsystem can request that cached items be discarded and their memory freed for other uses. One of the recurring topics at the [2013 Linux Storage, Filesystem, and Memory Management Summit](<https://lwn.net/Articles/LSFMM2013/>) was the need to improve the shrinker interface. The [proposed replacement](<https://lwn.net/Articles/550263/>) is out for review, so it seems like time for a closer look. 

#### A new shrinker API

In current kernels, a cache would implement a shrinker function that adheres to this interface: 

```
#include <linux/shrinker.h>
    
        struct shrink_control {
    	gfp_t gfp_mask;
    	unsigned long nr_to_scan;
        };
    
        int (*shrink)(struct shrinker *s, struct shrink_control *sc);
```

The `shrink()` function is packaged up inside a `shrinker` structure (along with some ancillary information); the whole thing is then registered with a call to `register_shrinker()`. 

When memory gets tight, the `shrink()` function will be called from the memory management subsystem. The `gfp_mask` will reflect the type of allocation that was being attempted when the `shrink()` call was made; the shrinker should avoid any actions that contradict that mask. So, for example, if a `GFP_NOFS` allocation is in progress, a filesystem shrinker cannot initiate filesystem activity to free memory. The `nr_to_scan` field tells the shrinker how many objects it should examine and free if possible; if, however, `nr_to_scan` is zero, the call is really a request to know how many objects currently exist in the cache. 

The use of a single callback function for two purposes (counting objects and freeing them) irks some developers; it also makes the interface harder to implement. So, one of the first steps in the new shrinker patch set is to redefine the shrinker API to look like this: 

```
long (*count_objects)(struct shrinker *s, struct shrink_control *sc);
        long (*scan_objects)(struct shrinker *s, struct shrink_control *sc);
```

The roughly two-dozen shrinker implementations in the kernel have been updated to use this new API. 

The current shrinker API is not NUMA-aware. In an effort to improve that situation, the `shrink_control` structure has been augmented with a new field: 

```
nodemask_t nodes_to_scan;
```

On NUMA systems, memory pressure is often not a global phenomenon. Instead, some nodes will have plenty of free memory while others are running low. The current shrinker interface will indiscriminately free memory objects; it pays no attention to which NUMA node any given object is local to As a result, it can dump a lot of cached data without necessarily helping to address the real problem. In the new scheme, shrinkers should observe the `nodes_to_scan` field and only free memory from the indicated NUMA nodes. 

#### LRU lists

A maintainer of an existing shrinker implementation may well look at the new NUMA awareness requirement with dismay. Most shrinker implementations are buried deep within filesystems and certain drivers; these subsystems do not normally track their cached items by which NUMA node holds them. So it appears that shrinker implementations could get more complicated, but that turns out not to be the case. 

While looking at the shrinker code, Dave Chinner realized that most implementations look very much the same: they maintain a least-recently-used (LRU) list of cached items. When the shrinker is called, a pass is made over the list in an attempt to satisfy the request. Much of that code looked well suited for a generic replacement; that replacement, in the form of a new type of linked list, is part of the larger shrinker patch set. 

The resulting "LRU list" data structure encapsulates a lot of the details of object cache management; it goes well beyond a simple ordered list. Internally, it is represented by a set of regular `list_head` structures (one per node), a set of per-node object counts, and per-node spinlocks to control access. The inclusion of the spinlock puts the LRU list at odds with normal kernel conventions: low-level data structures do not usually include their own locking mechanism, since that locking is often more efficiently done at a higher level. In this case, putting the lock in the data structure allows it to provide per-node locking without the need for NUMA awareness in higher-level callers. 

The basic API for the management of LRU lists is pretty much as one might expect: 

```
#include <linux/list_lru.h>
    
        int list_lru_init(struct list_lru *lru);
        int list_lru_add(struct list_lru *lru, struct list_head *item);
        int list_lru_del(struct list_lru *lru, struct list_head *item);
```

A count of the number of items on a list can be had with `list_lru_count()`. There is also a mechanism for walking through an LRU list that is aimed at the needs of shrinker implementations: 

```
unsigned long list_lru_walk(struct list_lru	*lru, 
    			        list_lru_walk_cb isolate,
    				void *cb_arg,
    				unsigned long nr_to_walk);
        unsigned long list_lru_walk_nodemask(struct list_lru *lru, 
        	     	  	        list_lru_walk_cb isolate,
    				void *cb_arg,
    				unsigned long nr_to_walk,
    				nodemask_t *nodes_to_walk);
```

Either function will wander through the list, calling the `isolate()` callback and, possibly, modifying the list in response to the callback's return value. As one would expect, `list_lru_walk()` will pass through the entire LRU list, while `list_lru_walk_nodemask()` limits itself to the specified `nodes_to_walk`. The callback's prototype looks like this: 

```
typedef enum lru_status (*list_lru_walk_cb)(struct list_head *item, 
        	    	 	    			spinlock_t *lock,
    						void *cb_arg);
```

Here, `item` is an item from the list to be examined, `lock` is the spinlock controlling access to the list, and `cb_arg` is specified by the original caller. The return value can be one of four possibilities, depending on how the callback deals with the given `item`: 

  * `LRU_REMOVED` indicates that the callback removed the item from the list; the number of items on the list will be decremented accordingly. In this case, the callback does the actual removal of the item. 

  * `LRU_ROTATE` says that the given item should be moved to the ("most recently used") end of the list. The LRU list code will perform the move operation. 

  * `LRU_RETRY` indicates that the callback should be called again with the same item. A second `LRU_RETRY` return will cause the item to be skipped. A potential use for this return value is if the callback notices a potential deadlock situation. 

  * `LRU_SKIP` causes the item to be passed over with no changes. 

With this infrastructure in place, a lot of shrinker implementations come down to a call to `list_lru_walk_nodemask()` and a callback to process individual items. 

#### Memcg-aware LRU lists

While an improved shrinker interface is well worth the effort on its own, much of the work described above has been driven by an additional need: better support for memory control groups (memcgs). In particular, memcg developer Glauber Costa would like to be able to use the shrinker mechanism to free only memory that is associated with a given memcg. All that is needed to reach this goal is to expand the LRU list concept to include memcg awareness along with NUMA node awareness. 

The result is a significant reworking of the LRU list API. What started as a simple list with some helper functions has now become a two-dimensional array of lists, indexed by node and memcg ID. A call to `list_lru_add()` will now determine which memcg the item belongs to and put it onto the relevant sublist. There is a new function — `list_lru_walk_nodemask_memcg()` — that will walk through an LRU list, picking out only the elements found on the given node(s) and belonging to the given memcg. The more generic functions described above have been reimplemented as wrappers around the memcg-specific versions. At this point, the "LRU list" is no longer a generic data structure (though one could still use it that way); it is, instead, a core component of the memory management subsystem. 

#### Closing notes

A review of the current shrinker implementations in the kernel reveals that not all of them manage simple object caches. In many cases, what is happening is that the code in question wanted a way to know when the system is under memory pressure. In current kernels, the only way to get that information is to register a shrinker and see when it gets called. Such uses are frowned upon; they end up putting marginally related code into the memory reclaim path. 

The shrinker patch set seeks to eliminate those users by providing a different mechanism for code that wants to learn about memory pressure. It essentially hooks into the [vmpressure mechanism](<https://lwn.net/Articles/524742/>) to set up an in-kernel notification mechanism, albeit one that does not use the kernel's usual notifier infrastructure. Interested code can call: 

```
int vmpressure_register_kernel_event(struct cgroup *cg, void (*fn)(void));
```

The given `fn()` will be called at the same time that pressure notifications are sent out to user space. The concept of "pressure levels" has not been implemented for the kernel-side interface, though. 

Most of this code is relatively new, and it touches a fair amount of core memory management code. The latter stages of the patch set, where memcg awareness is added, could be controversial, but, then, it could be that developers have resigned themselves to memcg code being invasive and expensive. One way or another, most or all of this code will probably find its way into the mainline; the benefits of the shrinker API improvements will be nice to have. But the path to the mainline could be long, and this patch set has just begun, so it may be a while before it is merged.
