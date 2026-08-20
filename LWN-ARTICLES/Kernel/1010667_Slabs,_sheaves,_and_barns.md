---
title: "Slabs, sheaves, and barns"
url: https://lwn.net/Articles/1010667/
date: "February 24, 2025"
category: "Memory management-Slab allocators"
author: "By Jonathan Corbet February 24, 2025"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 24, 2025

The kernel's slab allocator is responsible for the allocation of small (usually sub-page) chunks of memory. For many workloads, the speed of object allocation and freeing is one of the key factors in overall performance, so it is not surprising that a lot of effort has gone into optimizing the slab allocator over time. Now that the kernel is [down to a single slab allocator](<https://lwn.net/Articles/974138/>), the memory-management developers have free rein to add complexity to it; the latest move in that direction is the [per-CPU sheaves patch set](<https://lwn.net/ml/all/20250214-slub-percpu-caches-v2-0-88592ee0966a@suse.cz>) from slab maintainer Vlastimil Babka. 

Many kernel developers interact with the slab allocator using functions like [`kmalloc()`](<https://docs.kernel.org/core-api/mm-api.html#c.kmalloc>), which can allocate objects of any (reasonable) size. There is a lower level to the slab allocator, though, that deals with fixed-size objects; it is used heavily by subsystems that frequently allocate and free objects of the same size. The curious can see all of the special-purpose slabs in their system by looking at `/proc/slabinfo`. There are many core-kernel operations that involve allocating objects from these slabs and returning them, so the slab allocator has gained a number of features, including NUMA awareness and bulk operations, to accelerate allocation and freeing. 

But, naturally, it's never fast enough. 

One of the keys to improved performance on today's multiprocessor systems is to avoid interaction between the CPUs whenever possible. Interactions lead to contention, locking delays, and cache-line bouncing, all of which hurt performance, but a CPU that can act like the rest of the system isn't there can go forward at full speed. This understanding has driven the adoption of per-CPU data structures across the kernel. The slab allocator already makes use of per-CPU data, but it still has enough cross-CPU interactions to slow things down. 

Sheaves are a concept introduced by Babka's patch set; in essence, they are a per-CPU cache of objects that can be handed out in response to allocation requests without the need to interact with any other CPUs in the system. By default, sheaves are disabled, but they can be enabled for a specific slab by setting a non-zero value in the new field `sheaf_capacity` in the [`kmem_cache_args`](<https://elixir.bootlin.com/linux/v6.13.3/source/include/linux/slab.h#L265>) structure passed to [`kmem_cache_create()`](<https://docs.kernel.org/core-api/mm-api.html#c.kmem_cache_create>). The value is the number of objects that should be cached in a single sheaf; the [patch adding sheaf usage to the maple-tree data structure](<https://lwn.net/ml/all/20250214-slub-percpu-caches-v2-10-88592ee0966a@suse.cz>) sets it to 32\. 

When sheaves are enabled, the allocator will maintain a sheaf with the given number of objects for each CPU. An allocation request will be satisfied from this sheaf whenever possible, and freed objects will be placed back into the sheaf if there is room. That turns allocation and free operations into purely local assignments that can be executed quickly; no locking (or even atomic operations) required. There is a second (backup) sheaf maintained for each CPU as well; when the main sheaf is found to be empty, an object will be allocated from the backup sheaf instead. If the main sheaf is full when an object is freed, that object will be placed into the backup sheaf if possible. 

When both sheaves are full, there will no longer be a place to stash a freed object with a simple assignment; that is where the "barn" comes in. The barn is simply a place to keep sheaves that are not currently being used for caching by any CPU; there is one barn for each NUMA node in the system. Once a CPU has filled its sheaves, it will try to place one in the barn; if a CPU's sheaves are empty, it will try to obtain a full one from the barn. In either case, this operation is slower, since locking is required to safely access the shared barn, but it is still faster than going all the way into the slab allocator. 

The barn holds two sets of sheaves — one for full sheaves, and one for empty sheaves. If a CPU is freeing a lot of objects, it can place its full sheaves into the barn and obtain empty ones to replace them. There is a limit to the number of sheaves the barn can hold; it is wired to ten each for full and empty sheaves in the current patch set. If a CPU tries to place a sheaf into a full barn, that sheaf will be freed, along with any objects it contains, back into the slabs they came from. 

As described so far, this series has the potential to greatly accelerate memory-allocation operations for workloads that allocate and free a lot of slab objects. But there are a couple of other features that are added later in the series to make sheaves more useful. 

One of those is an enhancement to [`kfree_rcu()`](<https://docs.kernel.org/core-api/kernel-api.html#c.kfree_rcu>), which will delay the freeing of an object until after a read-copy-update (RCU) grace period has passed, ensuring that the object no longer has any active references. A third per-CPU sheaf is maintained to hold objects freed with `kfree_rcu()`; once the sheaf fills, it is passed to the RCU subsystem for the grace-period wait. Once that has happened, an attempt will be made to put the sheaf back into the barn. 

The other addition is preallocation. There are many places in the kernel where memory must be allocated without delay, and certainly without blocking the allocating thread. There are also numerous code paths that cannot deal with an allocation failure in any reasonable way. In most of these cases, there is an opportunity to preallocate the needed memory before going into the more constrained code. The kernel has long maintained subsystems like [mempools](<https://www.kernel.org/doc/html/latest/core-api/mm-api.html#memory-pools>) to meet this need. 

But, if the kernel can go into critical code in the knowledge that it has a per-CPU sheaf full of objects available to it, a lot of these problems (and the need for mempools) go away. To that end, the series provides [a set of functions](<https://lwn.net/ml/all/20250214-slub-percpu-caches-v2-6-88592ee0966a@suse.cz>) for working with special-purpose sheaves. A call to `kmem_cache_prefill_sheaf()` will return a sheaf containing at least the requested number of objects, grabbing it out of the barn if possible. Then, `kmem_cache_alloc_from_sheaf()` can be used to allocate objects from the sheaf with guaranteed success for at least the requested number of objects. Other functions can be used to return a sheaf to the barn or to place more objects into a sheaf. These special-purpose sheaves act similarly to mempools, but they are intended to be short-lived, unlike mempools which usually exist for the life of the system. 

This series appears to be relatively uncontroversial, though perhaps developers are reserving their comments for the upcoming [Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), to be held in late March. Given the appetite for faster memory allocation and freeing, though, sheaves and barns seem likely to be added to the mix at some point.
