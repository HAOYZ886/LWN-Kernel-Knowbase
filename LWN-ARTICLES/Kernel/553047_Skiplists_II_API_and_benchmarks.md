---
title: "Skiplists II: API and benchmarks"
url: https://lwn.net/Articles/553047/
date: "June 12, 2013"
category: Skiplists
author: "June 12, 2013 This article was contributed by Andrew Shewmaker"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

June 12, 2013

This article was contributed by Andrew Shewmaker

A skiplist is composed of a hierarchy of ordered linked lists, where each higher level contains a sparser subset of the list below it. In [part one](<https://lwn.net/Articles/551896>), I described the basic idea of a skiplist, a little history of various attempts to use it in the Linux kernel, and Chris Mason's new [cache-friendly skiplist](<https://lwn.net/Articles/549470/>) for index ranges. This article will continue with a description of the current state of Chris's skiplist API and his future plans for it. I'll also discuss the performance of skiplists and rbtrees in a simple RAM test, as well as Chris's more interesting IOMMU comparison. 

#### Skiplist API

A skiplist can be declared and initialized to an empty state with lines like: 

```
#include <linux/skiplist.h>
    
        struct sl_list list;
        sl_init_list(&list, GFP_ATOMIC);
```

Once the list exists, the next step is to populate it with data. As is shown in [the data structure diagram](<https://lwn.net/Articles/552088/>), each structure to be placed in the list should embed an `sl_slot` structure; pointers to this embedded structure are used with the skiplist API. 

Insertion into the skiplist requires the programmer to get a "preload token" — `skiplist_preload()` ensures that the necessary memory is available and disables preemption. With the token in hand, it's possible to actually insert the item, then re-enable preemption. Preloading helps avoid the need for atomic allocations and also to minimize time spent inside a leaf's lock during insertion. The preload function takes a pointer to a skiplist and a "get free page" mask describing the type of allocation to be performed, and it returns an integer token to be used later: 

```
int skiplist_preload(struct sl_list *list, gfp_t gfp_mask);
```

Note that preemption is disabled by `skiplist_preload()` and must not be re-enabled during insertion because the function is holding an RCU read lock and working with per-CPU data structures. 

The function that actually adds an item to the list, `skiplist_insert()`, is called with that list, a slot to be inserted, and a token returned by `skiplist_preload()`: 

```
int skiplist_insert(struct sl_list *list, struct sl_slot *slot, 
    			int preload_token);
```

Here's an example insertion into a skiplist: 

```
int preload_token, ret;
        preload_token = skiplist_preload(skiplist, GFP_KERNEL);
    
        if (preload_token < 0)
        	return preload_token;
    
        ret = skiplist_insert(skiplist, slot, preload_token);
        preempt_enable();
```

Deletion only requires one function call, though it is implemented in two phases if a leaf becomes empty. In that case, the leaf is marked "dead," then it is unlinked from the skiplist level by level. In either case, it returns the slot pointer of what it deleted from the list. 

```
struct sl_slot *skiplist_delete(struct sl_list *list, unsigned long key,
               	       		    unsigned long size);
```

Adding or removing a key to/from an existing leaf is simple and only requires a lock at the leaf. However, if a leaf is created or destroyed, then more locking is required. Leaves with higher levels require locks to be taken on neighboring nodes all the way down to level zero so that everything can be re-linked without having a neighbor being deleted out from under. The list of affected leaves is kept track of in a temporary `sl_node` list referred to as a cursor. (Chris is reworking his code to [get rid of cursors](<http://thread.gmane.org/gmane.linux.file-systems/75120/focus=75206>)). The best-case scenario is a modification at level zero where only a couple of locks are required. Both the preallocation and the insertion code are biased in favor of creating a level-zero leaf. Regardless, the locking is only required for a small window of time. 

Unlike an rbtree, rebalancing of the skiplist is not required, even when simultaneous insertions and deletions are being performed in different parts of the skiplist. 

A specialized insertion function is provided that finds a free index range in the skiplist that is aligned and of a given size. This isn't required by filesystems, but Chris implemented it so that he could directly compare rbtrees to skiplists in the IOMMU code. The IOMMU requires this functionality because each PCIE device's domain requires an aligned range of memory addresses. 

Calls to `skiplist_insert_hole()` take a hint of where a hole might be inserted, and must be retried with a new hint if the return value is `-EAGAIN`. That error return happens when simultaneous holes are being created and the one you hinted at was good, but was stolen before you could use it. On successful insertion, the slot passed in is updated with the location of the hole. 

```
int skiplist_insert_hole(struct sl_list *list, unsigned long hint,
        			     unsigned long limit, unsigned long size, unsigned long align,
        			     struct sl_slot *slot, gfp_t gfp_mask);
```

Tearing down a whole skiplist requires a fair amount of work. First free the structures embedding the slots of each leaf, then use `sl_free_leaf()`, and finally, zero the pointers in the head of the skiplist. Wrappers around `container_of()` for obtaining the leaf embedding a node or the structure embedding a slot are provided by `sl_entry(ptr)` and `sl_slot_entry(ptr, type, member)`, respectively. Comments in the code indicate future plans to add skiplist zeroing helpers, but for now you must roll your own as Chris did for his IOMMU patch. 

Here's a generic example of destroying a skiplist: 

```
struct sl_node *p;
        struct sl_leaf *leaf;
        struct sl_slot *slot;
        struct mystruct *mystruct;
    
        sl_lock_node(skiplist->head);
        p = skiplist->head->ptrs[0].next;
        while (p) {
    	    leaf = sl_entry(p);
    	    for (i = 0; i < leaf->nr; i++) {
    		    slot = leaf->ptrs[i];
    		    mystruct = sl_slot_entry(slot, struct mystruct, slot);
    		    free_mystruct_mem(mystruct);
    	    }
    	    p = leaf->node.ptrs[0].next;
    	    sl_free_leaf(leaf);
        }
    
        memset(skiplist->head->ptrs, 0, sl_node_size(SKIP_MAXLEVEL));
        sl_unlock_node(skiplist->head);
```

Chris considered including slot iterators equivalent to `rb_next()` and `rb_prev()`, but decided against it because of the overhead involved in validating a slot with each call. Instead, `skiplist_next()` and `skiplist_prev()` are leaf iterators that allow a caller to more efficiently operate on slots in bulk. Chris hasn't posted the updated API yet, but it seems likely that the iterators will resemble the existing `sl_next_leaf()` and friends. 

Calls to `sl_first_leaf()` and `sl_last_leaf()` return pointers to the first and last entries of the skiplist. The `sl_next_leaf()` call is a little different in that you must provide it with an `sl_node` (embedded in your current leaf), and since each node potentially has many next entries, you must also provide the level `l` you want to traverse. 

```
struct sl_leaf *sl_first_leaf(struct sl_list *list);
        struct sl_leaf *sl_last_leaf(struct sl_list *list);
        struct sl_leaf *sl_next_leaf(struct sl_list *list, struct sl_node *p, int l);
```

Since this skiplist implementation focuses on index ranges (or extents) defined by key and size parameters, it can provide search functions. This is in contrast to rbtrees--they are more diverse, so users must roll their own search functions. Each of the skiplist search functions needs to be passed a pointer to the skiplist, the key you are looking for, and the slot size (the number of extents in a leaf). If successful, they return a pointer to the slot matching the key. 

```
struct sl_slot *skiplist_lookup(struct sl_list *list, unsigned long key,
    				    unsigned long size);
        struct sl_slot *skiplist_lookup_rcu(struct sl_list *list, unsigned long key,
        				        unsigned long size);
```

The first, `skiplist_lookup()`, is appropriate for when a skiplist is experiencing high read/write contention. It handles all the locking for you. It protects the skiplist with [read-copy-update (RCU)](<https://lwn.net/Articles/262464/>) while it finds the correct leaf and then it protects the leaf with a spinlock during a binary search to find the slot. If no slot corresponds to the key, then a `NULL` pointer is returned. 

If skiplist contention is low or you need more control, then use the second variant. Before calling `skiplist_lookup_rcu()`, you must call `rcu_read_lock()` and you must take care of details such as reference counting yourself. The search for the leaf uses the same helper function as `skiplist_lookup()`, but the leaf spinlock is not held. Instead, it depends on the skiplist's RCU read lock being held to also protect the slots in a leaf while it performs a sequential search. This search is sequential because Chris does not do the copy part of RCU. He does order the operations of insertion/deletion to try to make the sequential search safe, and that should usually work. However, it might not return the slot of interest, so it is the responsibility of the caller to verify the key of the returned slot, and then call `skiplist_lookup_rcu()` again if it the returned slot's key doesn't match the key being searched for. 

Chris elaborated on his future plans for the API in a private email: 

In terms of changes coming to the patches, the biggest will be in the insert code. Right now skiplist_insert does the search, cursor maintenance, and the insert, but that won't work for XFS because they need more control over the `EEXIST` condition. 

It'll get split out into search and insert steps the caller can control, and you'll be able to call insert with just a locked leaf from any level... 

The searching API will also improve, returning both the leaf and the slot. This allows skiplist versions of rb_next() and rb_prev(). 

The skiplist code also indicates that there is work to be done to make lockdep understand Chris's skiplist locking. It needs to be taught that holding multiple locks on the same level of a skiplist is allowed as long as they are taken left to right. 

#### Testing

In addition to the IOMMU comparison between rbtrees and skiplists that Chris posted numbers for, his patch also includes a simple RAM-only comparison in the form of a kernel module called `skiplist_test`. I tested 100,000 items for 100,000 rounds with multiple numbers of threads. 

This table shows the results: 

> ADT | Threads | Fill  
> Time  
> (ms) | Check  
> Time  
> (ms) | Delete  
> Time  
> (ms) | Avg. Thread  
> Time  
> (s)  
> ---|---|---|---|---|---  
> rbtree | 1 | 37 | 9 | 12 | 0.752  
> skiplist-rcu | 1 | 18 | 15 | 23 | 2.615  
> rbtree | 2 | 36 | 8 | 12 | 2.766  
> skiplist-rcu | 2 | 19 | 19 | 27 | 2.713  
> rbtree | 4 | 36 | 11 | 10 | 6.660  
> skiplist-rcu | 4 | 23 | 24 | 21 | 3.161  
  
These results show skiplists beating rbtrees in fill time, but losing on check and delete times. The skiplist average thread time is only slightly better with two threads, and beats rbtree soundly with four threads (they take half the time). However, rbtree wins the single threaded case, which surprises Chris because it doesn't match what he sees in user-space testing. He told me, ""Most of the difference is the cost of calling spin_lock (even single threaded)."" 

The more interesting numbers are from Chris's IOMMU comparison. Even though he is mostly interested in using skiplists for Btrfs extents, he chose to use the IOMMU because it is easier to isolate the performance of the two data structures, which makes it both easier for non-Btrfs people to understand and more meaningful to them. He also says, ""... with the IOMMU, it is trivial to consume 100% system time on the rbtree lock."" The rbtree lock is, in effect, a global lock held once at the start and once at the end of an IO. 

Chris kept the basic structure of the IOMMU code so that he could compare skiplists to rbtrees. He was not trying to design a better IOMMU that looked for free ranges of addresses differently or fix the IOMMU contention, though he told me he would work with David Woodhouse on a proper solution that tracks free extents later this year. 

His benchmarks were run on a single socket server with two SSD cards. He used a few [FIO](<http://freecode.com/projects/fio>) jobs doing relatively large (20MB) asynchronous/direct IOs with 16 concurrent threads and 10 pending IOs each (160 total). Here are his results for streaming and random writes: 

```
Streaming writes
    IOMMU off  2,575MB/s
    skiplist   1,715MB/s
    rbtree     1,659MB/s
    
    Not a huge improvement, but the CPU time was lower.
    
    [...]
    
    16 threads, iodepth 10, 20MB random writes
    IOMMU off  2,548MB/s
    skiplist   1,649MB/s
    rbtree        33MB/s
```

The existing rbtree-based IOMMU slows streaming writes down 64.4% of the maximum, and the skiplist's throughput is slightly better at 66.6% while using less CPU time. Evidently the skiplist's advantages in concurrency and in maintaining a balanced overall structure only give it a modest advantage in the streaming write case. However, random writes cause rbtree performance to only achieve 1.3% of the maximum throughput. In this case, a skiplist fares much better, dropping only to 64.7% of the maximum because different threads can hold locks simultaneously while in different parts of the skiplist and it doesn't need to go through a costly rebalancing operation like the rbtree. 

```
16 threads, iodepth 10, 20MB random reads
    IOMMU off  2,861MB/s (mostly idle)
    skiplist   2,484MB/s (100% system time)
    rbtree        99MB/s (100% system time)
    
    ... lowering the thread count did improve the rbtree performance, but the
    best I could do was around 300MB/s ...
```

Reads are easier than writes, and we could expect streaming read results to all be close and relatively uninteresting. Certainly both the rbtree and skiplist do better at random reads than random writes. In fact, the skiplist achieves higher throughput for random reads than it does for streaming writes although it has to work hard to do so. And in case anyone thought the thread count was particularly unfair for rbtree in these tests, Chris points out that the best he got for random IOs with rbtree was around 300MB/s. That's still only 10% of the maximum throughput. Furthermore, Chris noted that all of the CPU time spent in the skiplist was in `skiplist_insert_hole()`, which isn't optimized. 

In a recent discussion on the Linux filesystems mailing list, Mathieu Desnoyers proposed another data structure that he is calling [RCU Judy arrays](<http://thread.gmane.org/gmane.linux.file-systems/75120>). They can't be compared with skiplists just yet since the Judy arrays are only implemented in user space so far, but the competition between the two ideas should improve them both. 

Even though there are plenty of opportunities for refinement, this is a promising start for a cache-friendly skiplist for the Linux kernel. It should provide better performance for any subsystem that has high levels of contention between concurrent accesses of its rbtrees: various filesystem indexes, virtual memory areas (VMAs), the high-resolution timer code, etc. CPU schedulers will probably not see any benefit from skiplists because only one thread is making the scheduling decision, but perhaps multiqueue schedulers for the network or block layer might in the case where they have one queue per NUMA node.
