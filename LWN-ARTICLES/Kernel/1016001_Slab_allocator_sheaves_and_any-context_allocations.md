---
title: "Slab allocator: sheaves and any-context allocations"
url: https://lwn.net/Articles/1016001/
date: "April 1, 2025"
category: "BPF-Memory management; Memory management-Slab allocators"
author: "By Jonathan Corbet April 1, 2025 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 1, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The kernel's slab allocator is charged with providing small objects on demand; its performance and reliability are crucial for the functioning of the system as a whole. At the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit, two adjacent sessions in the memory-management track dug into current work on the slab allocator. The first focused on the new sheaves feature, while the second discussed a set of allocation functions that are safe to call in any context. 

#### Ask sheaves

Sheaves are a new caching layer for the slab allocator that is intended to increase performance for heavy users; see [this article](<https://lwn.net/Articles/1010667/>) for a detailed description of how they work. Vlastimil Babka, the maintainer of the slab allocator, led the first session to discuss this feature and its future. He started by saying that some workloads are experiencing slowdowns as a result of a lot of slab allocations; some of that is explainable by how the slab allocator must track free objects within folios using an embedded free list. The folios containing the slabs must also be tracked, and maintaining NUMA locality adds another level of complexity. 

[![\[Vlastimil Babka\]](https://static.lwn.net/images/conf/2025/lsfmm/VlastimilBabka-sm.png)](<https://lwn.net/Articles/1016004/>) Sheaves bypass much of that work by maintaining a simple per-CPU cache in the form of two sheaves (arrays of free objects) for each CPU. It is currently an opt-in feature, but he wondered if it should just be enabled for all slabs. The caching makes freeing objects cheap, and there is integration with the read-copy-update system to reduce the cost of [`kfree_rcu()`](<https://elixir.bootlin.com/linux/v6.13.7/source/include/linux/rcupdate.h#L1034>). The allocator can also provide pre-filled sheaves on request that can be used to guarantee allocations in restricted contexts. This last feature comes nearly for free since, with luck, the allocator will have a sheaf with sufficient objects just sitting there when the request arrives. 

In summary, he said, the initial results from the sheaves layer are promising, but its interaction with slab debugging needs improvement. One question he had was: how much of a problem is the relative lack of NUMA awareness? Much of the complexity of the full object-freeing path in the slab allocator is the pains it takes to return freed objects to the proper NUMA node. Sheaves short all of that logic out and simply store free objects in the local CPU's sheaf, with the result that there will be more cross-node allocations. On systems with high cross-node access costs, that could perhaps change the performance characteristics of some workloads. For maple trees, which are the first user of sheaves, the allocations are used system-wide, so NUMA locality is less of a concern. 

He wondered again whether sheaves should be enabled for all users, though he worried that such a move might turn the current slab allocator into something more like its predecessor, which was recently removed. Perhaps, he suggested, sheaves could replace some of the other caching currently done in the slab allocator, maybe even to the point of replacing the current free-object tracking entirely. That would bring some challenges; without the free-object tracking, the allocator would need another way to recognize that a given slab has been entirely freed and can be returned to the page allocator, for example. 

As the session came to an end, Rik van Riel said that lock contention can be a real problem in the slab allocator now, and that sheaves look like they could help. Hoang Nhat Pham agreed, saying that he works with systems that have a large number of CPUs but only one NUMA node, leading to a lot of lock contention that would be improved by the per-CPU cache. 

#### `kmalloc()` for any context

Alexei Starovoitov then took over to run a joint session with the BPF track on work toward creating a version of `kmalloc()` (which is part of the slab allocator) that can be called from BPF programs in any context. Some BPF attachment points, including tracepoints, can be invoked from any context, even non-maskable interrupts (NMIs). The system's freedom to satisfy allocation requests can be highly constrained in those contexts, so care must be taken. The BPF subsystem has, for a few years, used a custom allocator to fill this need, but there is a desire to reduce the number of allocators (and object caches) in the kernel. See [this article](<https://lwn.net/Articles/1014220/#bpf>) for an overview of his proposed solution. 

[![\[Alexei
Starovoitov\]](https://static.lwn.net/images/conf/2025/lsfmm/AlexeiStarovoitov-sm.png)](<https://lwn.net/Articles/1016006/>) Starovoitov began by thanking Babka for [the 2024 LSFMM+BPF session on the slab allocator](<https://lwn.net/Articles/974138/>), without which he would not have had the courage to attempt this work. There are, he said, a lot of wrappers around the slab allocator in the kernel; there are different excuses for the existence of each. For BPF, the primary excuse was the need to be able to allocate in any context; performance is also important, but it is a secondary concern. 

Eleven years ago, he said, only the tracing and networking subsystems supported BPF programs. Tracing programs did not allocate memory, and the networking programs were only called in process context, so memory allocation was not a concern. Over time, though, the need for memory allocations in BPF programs grew; it was addressed initially with the BPF "maps" feature. Maps can have preallocated memory, but each map might end up preallocating 10KB of memory. That makes for a fast solution, but also leads to a lot of wasted memory. There have been some attempts to improve the situation, but they tend not to work well and are not heavily used. 

So the BPF memory allocator was added. It provides per-CPU buckets (to allocate objects of different sizes) like `kmalloc()` does, but it was designed to be NMI-safe and fast. At its core is a function called `bpf_mem_alloc()` to do the actual allocations. It was not as wasteful as map preallocation, but there is still a lot of memory that just sits in its caches. Some of that waste could be addressed by merging caches the way the slab allocator does — but that is just yet another step along the path of reimplementing features that the slab allocator already has. 

So it would be nice to be able to just use `kmalloc()`, but that allocation path is complex and heads into increasingly slow paths when needed to satisfy a request. The slowest steps can involve obtaining blocks from the zone allocator or waking the `kswapd` thread. In the wrong contexts, that path could lead to kernel deadlocks; using it would be, he said, ""a ticking time bomb"". 

So, instead, he has added `try_alloc_pages()`, which handles the lower-level page-allocation part of the task. It follows the same allocation path but stops short of taking actions that could deadlock the system. It is thus more likely to fail than other allocation paths. Most of the needed logic, he said, was already in place, there were only a few changes needed. Van Riel wondered, since conditional locking is used in this path, what happens if an attempt to take a lock to charge an allocation to a control group fails? In that case, Starovoitov answered, the entire allocation will fail. 

`try_alloc_pages()`, as can be seen from its name, is a page-level allocator; the next step is to create an all-context version of `kmalloc()` as well. That should be fairly straightforward, involving mostly mechanical changes, he said, but he is considering relaxing the node-locality requirements that `kmalloc()` upholds. It is better to allocate something than to fail the request because the only available memory is on a remote node. If the slab allocator needs more memory to satisfy a request, it will use `try_alloc_pages()` to attempt to allocate it. The code is written, he said, but is not yet ready for review. 

Van Riel asked how freeing works in this new interface; Starovoitov answered that, if it is not possible to obtain the needed locks to properly free a memory object, that memory will be added locklessly to a linked list instead, and the next non-atomic free call will take care of it. 

When this code was initially posted, memory-management subsystem maintainer Andrew Morton expressed opposition to it. He had been quiet during this session until the end, when Starovoitov asked whether that silence meant that his concerns had been addressed. Morton answered that he is still not entirely happy about this work, which addresses use cases that are not at all important relative to the core page allocator. It adds some run-time and maintenance overhead, he said, that would be best placed elsewhere so that only its users pay the cost. Starovoitov said that there is no measurable performance impact on the page allocator; Morton expressed skepticism and said that maintainability is important too, but seemed to grudgingly accept this change. 

The `try_alloc_pages()` changes [landed in the mainline](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aa918db707fb>) during the 6.15 merge window. 

Starovoitov has posted [the slides from this session](<https://github.com/4ast/docs/blob/main/Reentrant%20kmalloc.pdf>).
