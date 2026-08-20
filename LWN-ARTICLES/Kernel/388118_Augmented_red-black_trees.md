---
title: "Augmented red-black trees"
url: https://lwn.net/Articles/388118/
date: "May 18, 2010"
category: "Red-black trees"
author: "By Jonathan Corbet May 18, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 18, 2010

[Red-black trees](<http://lwn.net/Articles/184495/>) (rbtrees) are a highly-optimized data structure used in a number of places in the kernel. With an rbtree, a kernel programmer can quickly locate data structures corresponding to a specific value; all that is needed is to store data structures with the value of interest as the key. Some fuzzier sorts of matches can be hard to do with rbtrees; consider, for example, the case of finding the lowest-valued node which overlaps with a given range of values. Venkatesh Pallipadi recently encountered this problem while trying to improve the functioning of the page attribute table (PAT) support for the x86 architecture. Rather than give up on rbtrees, he chose to enhance that data structure to meet a wider range of needs. 

Venkatesh's patch (which was one of the first things merged for 2.6.35) implements the concept of "augmented rbtrees." Such a tree works very much like an ordinary rbtree, with the exception that it keeps additional information in each node. That information, almost certainly, is a function of any child nodes in the tree - the maximum key value among all children, for example. Since users of rbtrees must write their own search functions anyway, they can easily take advantage of this extra information to optimize searches. 

Users of augmented rbtrees must define an `augment_cb()` callback with this prototype: 

```
void (*augment_cb)(struct rb_node *node);
```

When the tree is initialized, the callback should be stored in its root node: 

```
struct rb_root my_root = RB_AUGMENT_ROOT(my_augment_cb);
```

Thereafter, the `augment_cb()` callback will be invoked whenever the value of one (or both) of a node's children might have changed. The callback can then update the node's additional information to match the new tree topology. The callback will be invoked from insert and delete operations - anything which might change the tree - so rbtree users should ensure that nodes are in a consistent state before inserting them. 

Callbacks are _not_ called recursively up the tree. So if a change to a node's augmented value might ripple upward, the `augment_cb()` callback must work its way up the tree and make the requisite updates. Note that a recursive call on the parent node is probably not a good idea unless the tree is known to be extremely shallow. 

As of this writing, the PAT code is the only in-tree user of this functionality, but others seem likely to appear now that this feature is globally available.
