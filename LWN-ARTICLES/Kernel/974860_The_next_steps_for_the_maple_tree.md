---
title: The next steps for the maple tree
url: https://lwn.net/Articles/974860/
date: "May 27, 2024"
category: Maple trees
author: "By Jonathan Corbet May 27, 2024 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 27, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The [maple tree data structure](<https://lwn.net/Articles/845507/>) was added during the 6.1 development cycle; since then, it has taken its place at the core of the kernel's memory-management subsystem. Unsurprisingly, work on maple trees is not yet done. Maple-tree maintainer Liam Howlett ran a session in the memory-management track of the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) to discuss the current state of the maple tree and which features can be expected next. 

Howlett has a backlog of requested features that seems likely to keep him busy for some time. Some of them are internal to the data structure itself: 

  * There is a desire for a fast way to get a count of the number of null entries within a node. 
  * "Dense nodes", which contain more pointers per node. 
  * The removal of "big nodes", which are a special structure used when nodes are rebalanced or split. Among other things, removing them will help to improve support for singleton ranges — process IDs, for example. 
  * Finally, he plans to implement index compression. 

With regard to externally visible features: 

  * The ability to search marks and tags is at the top of the to-do list. That would allow searching a tree for entries with, for example, the "dirty" bit set. 
  * The ability to prune trees under memory pressure would help the system overall; it could be used with the cache that holds shadow page-table entries for evicted pages. 
  * Filesystem users would benefit from 64-bit indices on 32-bit architectures. 
  * A contiguous iterator that would iterate over a range only as long as there are no gaps. 
  * "Big dense nodes" were described as a large list that could hold up to 4K singleton items. 

Overall, he said, he is trying to get maple trees to the point that they can match the features provided by [XArrays](<https://docs.kernel.org/core-api/xarray.html>). The maple tree should be able to do the same things with better performance; once the features are there, it should be possible to implement the XArray interface and switch users without anybody else having to even be aware of it. 

Howlett said that the maple tree is getting more users, and he is seeing some common errors when code is converted over. It is possible to use external locks to serialize access to a maple tree, he said, and some users do it, but it is better avoided if possible. He cautioned that anybody using read-copy-update (RCU) read locks should be aware that the lock protects a maple-tree node from being freed, but not necessarily the data contained within that node. 

Users of the generic storage API were encouraged to wrap it with a typed interface so that the compiler can catch mistakes. Developers converting from an XArray often are surprised when [`mas_next()`](<https://elixir.bootlin.com/linux/v6.9.1/source/lib/maple_tree.c#L5741>) fails to return the first entry. Its job is to get the _next_ entry; to start at the beginning, [`mas_find()`](<https://elixir.bootlin.com/linux/v6.9.1/source/lib/maple_tree.c#L6048>) should be used instead. 

In general, he said, he is working toward the addition of a type-safe interface and moving away from `void *` pointers. Eventually there will be a `DEFINE_MAPLE_TREE()` macro that creates a tree handling objects of a given type. 

As usage has grown, the maple tree structure has encountered a number of challenges. Tracking of virtual memory areas (VMAs) is one of those; he is trying to find ways to remove some of the complexity associated with special VMAs. One example is guard VMAs, which define a short range of no-access address space to catch overruns. If guard VMAs are in use, the total number of VMAs in the tree is doubled, which is expensive, but those guard VMAs are never really used. So Howlett is trying to find a way to mark guard regions directly in the maple tree and avoid allocating so many extra structures. 

Maple trees should eventually implement upper and lower limits, he said; that would be useful, for example, to implement restrictions on mapping the page at virtual address zero. Currently a maple tree will show gaps in areas that are not actually available for allocation. There are also some challenges in representing the [vDSO](<https://en.wikipedia.org/wiki/VDSO>) area. 

There were a few comments once Howlett finished. David Hildenbrand said that the kernel contains a lot of checks for gate VMAs, which are a special VMA used to represent the virtual-system-call page; it would be nice to find a way to represent them in the maple tree and remove those checks. Suren Baghdasaryan said that guard VMAs are one of the biggest allocation slabs on Android systems, so removing them would be a welcome optimization. The session wound down with a bit of discussion on the best way to identify guard VMAs within the kernel.
