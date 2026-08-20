---
title: A kernel skiplist implementation (Part 1)
url: https://lwn.net/Articles/551896/
date: "May 30, 2013"
category: Skiplists
author: "May 30, 2013 This article was contributed by Andrew Shewmaker"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

May 30, 2013

This article was contributed by Andrew Shewmaker

The Linux kernel manages sparse sets of index ranges for various subsystems. For instance, the I/O memory management unit (IOMMU) keeps track of the addresses of outstanding device memory mappings for each PCIE device's domain and tracks holes for new allocations. File systems cache pending operations, extent state, free extents, and more. A simple linked list fails to provide good performance for search, insertion, and deletion operations as it grows, so more advanced abstract data types like [red-black trees](<https://lwn.net/Articles/184495/>) (rbtrees) were implemented. The kernel's rbtrees generally provide good performance as long as they are only being accessed by one reader/writer, but they suffer when multiple readers and writers are contending for locks. 

There are variants of rbtrees that allow efficient [read-copy-update (RCU)](<https://lwn.net/Articles/262464/>) based reads, but not fine-grained concurrent writes. So, Chris Mason of Btrfs fame is developing a [skiplist implementation](<https://lwn.net/Articles/549470/>) that will allow file systems like Btrfs and XFS to perform many concurrent updates to their extent indexes. This article will describe the basic skiplist and what makes this skiplist variant cache-friendly. In part two, I'll describe the skiplist API and compare the performance of skiplists to rbtrees. 

#### Basic skiplist

William Pugh first described skiplists in 1990 as a probabilistic alternative to balanced trees (such as rbtrees) that is simpler, faster, and more space-efficient. A skiplist is composed of a hierarchy of ordered linked lists, where each higher level contains a sparser subset of the list below it. The size of the skiplist decreases at a constant rate for each higher level. For instance, if level zero has 32 elements and a probability _p_ =0.5 that an element appears in the next higher level, then level one has 16 elements, level two has eight, etc. The subset selected for the next higher level is random, but the quantity of items in the subset isn't random. 

By way of analogy, consider an old-fashioned printed dictionary with multiple search aids built into it. This diagram shows how the different aids are similar to levels of a skiplist: 

> ![\[Skiplist diagram\]](https://static.lwn.net/images/2013/skiplist/basicskiplist-dict.png)

At the highest level there are grooves on the edge of the book that mark sections for each letter, A-Z. The search can continue by looking at word prefixes centered at the top of each page. Next, guide words at the top left and right of each page show the span of words for each page. At the lowest level, a page is scanned word by word. A skiplist replicates something like this index structure in a digital form. 

#### History of kernel skiplists

Josh MacDonald investigated [cache-friendly skiplists](<http://sourceforge.net/projects/skiplist/>) on Linux 2.4 more than a decade ago. His skiplists are more space-efficient than rbtrees with more than 100 keys, searches are faster after 1,000 keys, and deletions and insertions at 10,000 keys. He also performed various concurrency experiments. When Josh posted an [RFC to the kernel mailing list](<http://lkml.indiana.edu/hypermail/linux/kernel/0109.2/1067.html>) he presented them as a ""solution waiting for the right problem to come along"". At the time, no discussion ensued, but Chris later became aware of Josh's work during his research. Unfortunately, Josh's implementation focused on reader/writer locks, which only allow one writer at a time, and Chris told me that he wanted to use RCU. Also, Josh's coding style differs from the kernel's and his macros make his code a little difficult to work with. 

Other well-known Linux kernel developers have experimented with skiplists as well. Con Kolivas initially thought they would be useful for his [BFS CPU scheduler](<https://lwn.net/Articles/460430/>). Ultimately, [BFS experienced worse behavior in general when using skiplists](<http://ck-hack.blogspot.com/2011/10/bfs-0413-bfs-benchmarks-and-more-cksort.html>), so Con returned to his previous list structure. This was due in part to the BFS scheduler's common case of having few tasks to schedule, so it didn't benefit from theoretically faster lookups for larger numbers of items in the list. Also, the scheduler process wasn't parallel itself, so it didn't benefit from a skiplist's more easily exploited concurrency. 

Andi Kleen recalled the discouraging results from his own [skiplist investigation](<http://thread.gmane.org/gmane.linux.kernel/1194885/focus=1194892>) to Con. He found that the variable number of pointers needed to link different levels of a skiplist made it difficult to achieve efficient use of memory and cache without limiting the size of the skiplist. 

In 2011, Chris asked Liu Bo to try replacing Btrfs's rbtree-based `extent_map` code, which maps logical file offsets to physical disk offsets, with a skiplist-based implementation. The mappings are read-mostly, until a random write workload triggers a copy-on-write operation. Liu created an [initial skiplist patch for Btrfs](<http://thread.gmane.org/gmane.comp.file-systems.btrfs/15150>) beginning with Con's implementation and adding support for concurrency with RCU. Results were mixed, and Chris's user-space experimentation led him to start work to make Liu's skiplists more cache-friendly. 

You may be saying to yourself, "I thought the whole point of Btrfs was to use B-trees for everything." Btrfs does use its copy-on-write B-trees for any structure read from or written to the disk. However, it also uses rbtrees for in-memory caches. Some rbtrees batch pending inode and extent operations. Other caches are: the extent state cache -- tracking whether extents are locked, damaged, dirty, etc.; the free space cache -- remembering free extents for quicker allocations; and the previously mentioned `extent_map`. In addition to the rbtrees, a radix tree manages the extent buffer cache, which is used like the page cache, but for blocks of metadata that might be larger than a page. 

All of these data structures have multiple threads contending for access to them and might benefit from skiplists, though the delayed operation trees and free space cache have the most contention. However, Chris's real target for this skiplist is some pending RAID5/6 parity logging code. It needs to enable "fast concurrent reads and writes into an exception store with new locations for some extents." Ideally, Chris hopes to make his skiplist general purpose enough for others to use. If skiplists can provide lockless lookups and be used for both buffer cache indexes and extent trees, then [ Dave Chinner would consider using them in XFS](<http://thread.gmane.org/gmane.linux.file-systems/74051/focus=74069>). 

#### Cache-friendly skiplist

When reading the following description of Chris Mason's skiplist implementation, keep in mind that it is a skiplist for range indexes, or extents. The extents being managed are not identified by a single number. Each has an index/key pointing to its beginning and a range/size. Furthermore, each element of the skiplist is composed of multiple extents, which will be referred to as slots. 

This new implementation of a cache-friendly skiplist is a bit more complicated than the picture of the basic skiplist may suggest; it is best examined in pieces. The first of those pieces is described by this diagram (a subset of [the full data structure diagram](<https://lwn.net/Articles/552088/>)): 

> ![\[skiplist head\]](https://static.lwn.net/images/2013/skiplist/cacheskiplist-head.png)

This figure shows a skiplist anchored by an `sl_list` structure pointing to the initial entry (represented by ` struct sl_node`) in the list. That `sl_node` structure has an array of pointers (called `ptrs`), indexed by level, to the head `sl_node_ptr` structures of each skiplist level. The `sl_node_ptr` structure functions like a typical kernel `list_head` structure, except that each level's head `sl_node_ptr->prev` is used, possibly confusingly, to point to the item with the greatest key at that level of the skiplist. All locking is done at the `sl_node` level and will be described in part two of this article. 

The skiplist grows from the head `sl_node_ptr` array on the right of the diagram above into an array of linked lists as shown below: 

> ![\[skiplist node pointers\]](https://static.lwn.net/images/2013/skiplist/cacheskiplist-node_ptr.png)

Note that none of the structures listed so far contain any keys or data, unlike traditional skiplists. That's because Chris's skiplist items are associated with more than one key. Another difference is that previous skiplist implementations lack `prev` pointers. Pugh's original skiplist didn't need something like `prev` pointers because it didn't support concurrency. MacDonald's skiplist does support concurrency, but its protocol uses context structures that combine parent/child pointer information with lock states. Mason's skiplists use a different concurrency protocol that uses `prev` pointers. 

A superficial difference between this skiplist and others is its apparent lack of `down` pointers (allowing movement from one level of index to the next), although they exist as the `ptrs[]` array in `sl_node`. While the pointer array name is unimportant, its position is, because different level nodes are created with different sizes of arrays. See the `ptrs` array at the end of `sl_node` and `sl_node` at the end of `struct sl_leaf` (which holds the actual leaf data): 

> ![\[Skiplist leaf\]](https://static.lwn.net/images/2013/skiplist/cacheskiplist-leaf.png)

The variable size of nodes could cause memory fragmentation if they were allocated from the same pool of memory, so Chris designed his skiplist with one slab cache for each level. This effectively addresses Andi Kleen's concerns mentioned earlier regarding wasting memory while keeping good performance. 

The way in which keys are handled is cache-friendly in a couple of ways. First, keys are not associated directly with individual items in this skiplist, but each leaf manages a set of keys. `SKIP_KEYS_PER_NODE` is currently set to 32 as a balance between increasing cache hits and providing more chances for concurrency. Second, an `sl_leaf` contains an array of `keys`. Technically a `keys` array is unnecessary, since the keys are part of the slots linked to a leaf. However, a search only needs the `keys` and not the rest of the `sl_slot` structures until the end of the search. So, the array helps skiplist operations avoid thrashing the cache. 

Each `sl_slot` is an extent defined by its `key` and `size` (index range). A developer can adapt the skiplist for their own use by embedding `sl_slot` into their own data structure. The job of reference counting belongs to the container of `sl_slot`. 

See [the full diagram](<https://lwn.net/Articles/552088/>) for a picture of how all the pieces described above fit together. With 32 keys per item, a maximum level of 32, and half the number of items in each higher level, Chris's skiplist should be able to efficiently manage around 137 billion extents (index ranges). 

The statistically balanced nature of the skiplist allows it to handle sparse sets of indexes in such a way that modifications avoid the need for an rbtree's rebalancing operations, and so they lend themselves to simpler concurrency protocols. Chris also designed his skiplist to be cache-friendly by having each element represent a set of keys/slots, keeping duplicates of the slot's keys in an array, and creating a slab cache for each level of the skiplist. Part two of this series will offer a description of the skliplist API and a discussion of the performance of skiplists vs. rbtrees.
