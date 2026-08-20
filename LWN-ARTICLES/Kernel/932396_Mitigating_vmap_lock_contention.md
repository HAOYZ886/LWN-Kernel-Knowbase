---
title: Mitigating vmap lock contention
url: https://lwn.net/Articles/932396/
date: "May 26, 2023"
category: "Memory management-Scalability"
author: "By Jonathan Corbet May 26, 2023 LSFMM+BPF"
---

By **Jonathan Corbet**  
May 26, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The "vmap area" is a range of kernel address space used when the kernel needs to virtually map a range of memory; among other things, memory allocations obtained from `vmalloc()` and loadable modules are placed there. At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Uladzislau Rezki, presenting remotely, explained a performance problem related to the vmap area and discussed possible solutions. 

The problem, he said, is that the vmap area is protected by three global spinlocks. The global free-space area is covered by `free_vmap_area_lock`, the tracking of mapped areas by `vmap_area_lock`, and the list of lazily freed areas by `purge_vmap_area_lock`. These locks, he said, can turn into a significant bottleneck on systems with a large number of CPUs. The `vmap_area_lock` controls access to a [red-black tree](<https://lwn.net/Articles/184495/>) that can be used to find an allocated area using an address within it. These areas can be seen by looking at `/proc/vmallocinfo`. The `free_vmap_area_lock`, instead, regulates access to free space and can experience high lock contention. 

[![\[Uladzislau
Rezki\]](https://static.lwn.net/images/conf/2023/lsfmm/UladzislauRezki-sm.png)](<https://lwn.net/Articles/932400/>) The allocation path has to acquire both `free_vmap_area_lock` (to find a free range) and `vmap_area_lock` (to mark that range as busy). The freeing path, instead, needs `vmap_area_lock` and `purge_vmap_area_lock`. This pattern means that the three areas cannot be accessed concurrently. Running some tests on a "super-powerful computer", Rezki measured a basic `vmalloc()` call as taking about 2µs when a single thread was running. With 32 threads calling `vmalloc()` simultaneously, that time grew to 50µs — 25 times greater. That slowdown is the result of contention on the vmap-area locks. 

The biggest problem, he said, is `vmap_area_lock`. This is partly due to a fair amount of fragmentation in the allocated areas, he said; the free and purge lists have fewer, larger areas and, as a result, less contention. Rezki proposed addressing this problem by adding a per-CPU cache; each CPU would pre-fetch some address space into its cache, then allocate pieces of that space to satisfy requests. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

An attendee pointed out that the problem of allocating vmap-area space looks similar to allocating user-space address space and asked whether the same infrastructure could be used for both. Rezki answered that user-space allocation is a bigger problem, so the solution is heavier, and optimized implementations are still in development. The real problem with the vmap area is the serialization of requests across CPUs, which is amenable to a simpler solution. 

Liam Howlett said that the `vmap_area_lock` is used for both allocation and freeing operations; if it could be avoided in one of the two paths, that could reduce contention. Rezki said that is true in theory, but that the bookkeeping has to be done somehow regardless. Howlett repeated that the problem is similar to the allocation of virtual-memory areas for user space. Memory-management developers should learn from each other, he said, rather than going off and doing their own things. 

Rezki moved on to the management of free space in the vmap area. When a range in that area is freed, the approach would be to convert the address into the appropriate per-CPU zone, lock that zone, and remove the allocation. Then the lazy-free zone could be locked, and the newly freed area added there. A separate context would occasionally drain that lazy list; in his patch set it is being drained to the global area for now. 

He concluded by asking what his next steps should be; the answer was to post patches and follow the usual process. He was asked for performance numbers, but had none available. When asked where this contention has been observed, he said it shows up on Android systems during video playback. The session ended with Michal Hocko suggesting that Rezki join his work with the efforts to improve user-space address allocation if possible.
