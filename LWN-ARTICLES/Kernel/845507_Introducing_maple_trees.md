---
title: Introducing maple trees
url: https://lwn.net/Articles/845507/
date: "February 12, 2021"
category: Maple trees
author: "February 12, 2021 This article was contributed by Marta Rybczyńska"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

February 12, 2021

This article was contributed by Marta Rybczyńska

Seen from outside, the internals of the Linux kernel appear to be stable, especially in subsystems like the memory-management subsystem. However, from time to time, developers need to replace an internal interface to solve a longstanding problem. One such issue is contention on the lock used to protect essential memory-management structures, including the page tables and virtual memory areas (VMAs). Liam Howlett and Matthew Wilcox have been developing a new data structure, called a "maple tree", to replace the data structures currently used for VMAs. This potentially big change in internal kernel structures has been recently [posted](<https://lwn.net/ml/linux-mm/20210112161240.2024684-1-Liam.Howlett@Oracle.com/#t>) for a review in a massive patch set.

Linux is a virtual-memory system. The address space for each process contains multiple virtual memory areas represented by [`vm_area_struct`](<https://elixir.bootlin.com/linux/v5.10.13/source/include/linux/mm_types.h#L306>) structures. Each VMA represents a contiguous block of address space and represents a range of memory of the same type, which may be anonymous memory (not backed by a file), a memory-mapped file, or even device memory. A virtual memory area, as seen from the process, is contiguous, while the supporting physical memory may not be. In addition, the address space contains holes between the VMAs; the kernel fills those empty blocks (leaving space for unmapped "guard" pages to catch buffer overflows) when it needs to add a new mapped area, for example when loading a library or in a response to an [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) call.

Almost anything one can do in the system involves memory, so the operations on the structures representing the VMAs must be fast. These operations include lookups (finding out which VMA corresponds to a given virtual address, finding out if the memory is mapped, or locating an empty gap for a new memory area), and modifications (growing a stack, for example).

VMAs are currently stored in a modified [red-black tree (rbtree)](<https://lwn.net/Articles/184495/>) with an additional, doubly-linked list that allows the kernel to traverse all of the VMAs in a process's address space. Kernel developers have been unhappy with this data structure for some time, for a number of reasons: rbtrees do not support ranges well, they are difficult to handle in a lockless manner (the balancing operation of the rbtree affects multiple items at the same time), and rbtree traversal is inefficient, which is why the additional linked list exists.

Operations on VMAs are protected by a lock (specifically a reader/writer semaphore), which is contained in [`struct mm_struct`](<https://elixir.bootlin.com/linux/v5.10.12/source/include/linux/mm_types.h#L389>). This lock was called `mmap_sem` until it was [renamed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/include/linux/mm_types.h?h=v5.11-rc6&id=da1c55f1b272f4bd54671d459b39ea7b54944ef9>) to `mmap_lock` for the 5.8 release in June 2020. This renaming was a part of an effort to wrap the lock in an API, hoping to ease its replacement in the future.

Users, especially those with threaded applications in large systems, often experience contention on this lock. The problem has been discussed by kernel developers numerous times, with at least [three sessions discussing it](<https://lwn.net/Articles/787629/>) at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LFSMM). The core of the problem is that the lock is required for many operations, including almost anything involving page tables and VMAs. There are other related structures that are effectively protected by `mmap_lock` (with the additional difficulty of lack of documentation of this fact). In addition to splitting the unrelated structures out from under `mmap_lock`, the developers were considering either using a structure that would allow lockless access to the VMAs or using some type of range lock. The maple tree was proposed as one of the solutions at that time, but at that time it was in an early state of development and the code was not available yet.

#### Introducing maple trees

Maple trees (the name might refer to the shape of the maple leaf, leading in multiple directions) differ in important aspects from rbtrees. They belong to the [B-tree](<https://en.wikipedia.org/wiki/B-tree>) family, so their nodes can contain more than two elements — up to 16 elements in leaf nodes, or ten elements in internal nodes. The use of B-trees also means that there is less need to create new nodes, as nodes may include empty slots that can be filled over time without additional allocations. Each node requires at most 256 bytes, which is a multiple of popular cache line sizes. The increased number of elements in a node and the cache-aligned size means fewer cache misses when traversing the tree.

The improved support for searching in maple trees also comes from the B-tree structure. In a B-tree, each node holds keys, called "pivots", that separate the node into subtrees. A subtree before a given key contains only values lower or equal to the key, while subtree after the key contains only values higher than the key (and lower than the next key).

Of course, maple trees were designed to work in a lockless manner, using read-copy-update (RCU). The maple tree is a generic structure that can be used in different kernel subsystems; the first usage is replacing the rbtrees and linked lists to manage VMAs. One of the authors, Liam Howlett, [explained](<https://blogs.oracle.com/linux/the-maple-tree>) the reasons for the design in a blog post.

Maple trees offer two APIs: the simple one and the advanced one. The simple API uses the `mtree_` prefix for its functions, with the main structure, `struct maple_tree`, defined as follows:

```
struct maple_tree {
    	spinlock_t    ma_lock;
    	unsigned int  ma_flags;
    	void __rcu    *ma_root;
        };
```

The static initializers are `DEFINE_MTREE(name)` and `MTREE_INIT(name, flags)`, with the latter taking a bitmask of flags from the two defined currently. `MAPLE_ALLOC_RANGE` indicates that the tree will be used to allocate ranges and that it should manage gaps between the allocations; `MAPLE_USE_RCU` activates the RCU mode for the case of multiple readers. Dynamic initialization is possible with the same flags using `mtree_init()`: 

```
void mtree_init(struct maple_tree *mt, unsigned int ma_flags);
```

Developers can free the whole tree with: 

```
void mtree_destroy(struct maple_tree *mt);
```

Three functions exist to add entries to the tree; `mtree_insert()`, `mtree_insert_range()`, and `mtree_store_range()`. The first two functions only add an entry if it does not previously exist, while the third function can overwrite an existing entry. They are defined as follows:

```
int mtree_insert(struct maple_tree *mt, unsigned long index,
    	void *entry, gfp_t gfp);
        int mtree_insert_range(struct maple_tree *mt, unsigned long first,
    	unsigned long last, void *entry, gfp_t gfp);
        int mtree_store_range(struct maple_tree *mt, unsigned long first,
    	unsigned long last, void *entry, gfp_t gfp);
```

`mtree_insert()` takes a pointer to the tree `mt`, the integer key of the entry `index`, the pointer to the entry `entry`, and the [memory allocation flags](<https://elixir.bootlin.com/linux/v5.10.14/source/include/linux/gfp.h#L230>) `gfp` for new tree nodes (if needed). `mtree_insert_range()` inserts a range from `first` to `last` with the given data `entry`. These functions return zero on success, and a negative value otherwise, including `-EEXIST` if the key already exists. Finally, `mtree_store_range()` takes the same arguments as `mtree_insert_range()`; the difference is that it replaces any existing entries for the corresponding keys.

Two functions exist to get an entry from the tree or remove an entry:

```
void *mtree_load(struct maple_tree *mt, unsigned long index);
        void *mtree_erase(struct maple_tree *mt, unsigned long index);
```

To read an entry, the developer should use `mtree_load()`, which takes a pointer to the tree `mt` and the key of the requested value `index`. The function returns a pointer to the entry, or NULL if the key was not found in the tree. The same syntax is used to remove an entry from the tree using `mtree_erase()`. It removes the given key from the tree and returns the associated entry, returning NULL if no such value was found.

There is more to the simple API than the above, including `mtree_alloc_range()` to allocate a range from the key space. The advanced API (which uses the `mas_` prefix) adds iterator functions for the traversal of the whole tree, or to access next and previous elements using a state variable. With this fine-grained control, developers can resume a search as needed. The API also allows developers to find empty areas or duplicate the tree.

#### Replacing the VMA rbtree (and more)

The patch set contains more than just the introduction of maple trees. It is worth noting that a big part of the set adds and updates tests, a welcome addition given the impact of the changes and the future importance of the new data structure.

In the 70 patches in this set, maple trees are used to replace rbtrees in all operations on VMAs, and one of the patches removes the usage of rbtrees from VMAs. Another part of the patch set leads to the removal of the VMA linked list. This removal requires modifications in various places in the kernel code that were using the VMA list directly: the architecture code, core dump and program loading routines, some device drivers, and of course the memory-management code. The patch set also [removes the VMA cache](<https://lwn.net/ml/linux-mm/20210112161240.2024684-20-Liam.Howlett@Oracle.com/>) (which tracks the VMAs that are most recently accessed by each process to speed lookup), as the implementation with maple trees is faster and the cache is no longer needed.

The cover letter includes some performance results, which are somewhat hard to interpret. Some microbenchmarks show a performance increase, and some (a smaller number) a decrease. Kernel compilation time is similar to that with the vanilla 5.10 kernel, with a few more instructions executed (probably linked to the added code). Howlett requested insights from developers into the results.

#### Current status and next steps

Maple trees are currently at the RFC stage, with limitations; the implementation does not support 32-bit systems or non-MMU platforms, for example. However, the code is functional and kernel developers may look into it to decide if it is the direction they want to go on the quest of removing `mmap_lock` (as the lock was not removed in this patch set). Based on the size of the patch set, it may take time until the review is finished.
