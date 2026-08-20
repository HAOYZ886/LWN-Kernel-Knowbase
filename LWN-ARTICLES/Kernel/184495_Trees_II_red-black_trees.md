---
title: "Trees II: red-black trees"
url: https://lwn.net/Articles/184495/
date: "June 22, 2006"
category: "Red-black trees"
author: 
---

This article is the somewhat delayed followup to [Trees I](<http://lwn.net/Articles/175432/>), which looked at the radix tree interface. Along with radix trees, the kernel contains an implementation of a data structure known as a "red-black tree." These trees (known as "rbtrees" within the kernel) are a form of semi-balanced binary tree. Each node in the tree contains a value and up to two children; the node's value will be greater than that of all children in the [![\[Red-black tree\]](https://static.lwn.net/images/ns/kernel/wikipedia-rbtree.png)](<http://en.wikipedia.org/wiki/Image:Red-black_tree_example.png>) "left" child branch, and less than that of all children in the "right" branch. Thus, it is possible to serialize a red-black tree by performing a depth-first, left-to-right traversal. 

Every node in a red-black tree is considered to be colored either red or black, with the root always being black. There is a somewhat complicated set of rules on how nodes should be colored, and, in particular, how the colors of the nodes should be used to make decisions on when and how to rebalance the tree. This article will not go into the details of the red-black tree mechanism, especially since that mechanism is well described by [the Wikipedia red-black tree article](<http://en.wikipedia.org/wiki/Red-black_tree>) (which is also the source of the image used here). Instead, we'll focus on how red-black trees are used in the Linux kernel. 

The complex rules for red-black trees do bring some advantages. Since it is a binary tree, a red-black tree can perform lookups in logarithmic time. If the tree is properly maintained, the longest path to a leaf node in the tree will never be more than twice as long as the shortest path - in other words, the tree is always in approximate balance. But the property which is arguably most useful in the kernel context is the fact that insertions and deletions are (1) fast, and (2) provably bounded in time. All the work that the kernel developers have put into reducing latencies would be wasted if a data structure were to simply go off for an indeterminate period of time rebalancing itself. Users of red-black trees pay a small lookup cost because the tree is not perfectly balanced, but, in return, they get fast, bounded insertion and deletion operations. A red-black tree can, thus, be indicated in situations where nodes come and go frequently. 

There are a number of red-black trees in use in the kernel. The anticipatory, deadline, and CFQ I/O schedulers all employ rbtrees to track requests; the packet CD/DVD driver does the same. The high-resolution timer code uses an rbtree to organize outstanding timer requests. The ext3 filesystem tracks directory entries in a red-black tree. Virtual memory areas (VMAs) are tracked with red-black trees, as are epoll file descriptors, cryptographic keys, and network packets in the "hierarchical token bucket" scheduler. 

The process of using a red-black tree starts by including `<linux/rbtree.h>`. This is one of the trickier kernel data structures to use, however. When designing a general data structure for a language like C, the developer must always decide how to include arbitrary types within the structure, and how to make comparisons between them. The person who implemented Linux rbtrees (the copyright in the code is to Andrea Arcangeli) made these decisions: 

  * Structures which are to be part of an rbtree must include a `struct rb_node` within them; there are no `void *` pointers to separate objects. This is a common way of implementing kernel data structures, and so will not surprise too many people. 

  * There is no "compare two objects" callback used in the rbtree code. Instead, users of rbtrees must, for all practical purposes, write the top-level search and insertion functions themselves, using lower-level rbtree primitives. As a result, using an rbtree is a bit more work, and the data structure is rather less opaque than our computer science teachers would have liked. What is gained in return, however, is a faster overall implementation without a bunch of indirect function calls in the hottest part of the tree traversal loops. 

It should also be remembered that an rbtree, like many other kernel data structures, implements no locking of its own. Any code which uses an rbtree must implement its own mutual exclusion to keep the tree from being corrupted. Usually, that locking will fit well with the scheme already being used by that code anyway, so there is no need for an independent locking mechanism. 

The root of a red-black tree has the type `struct rb_root`; a tree can be initialized to the empty state with a line like: 

```
struct rb_root the_root = RB_ROOT;
```

Assume, for a moment, that we have a red-black tree which is already full of interesting data. Traversal of that tree (which does not involve searching) is straightforward: 

```
struct rb_node *rb_first(struct rb_root *tree);
        struct rb_node *rb_last(struct rb_root *tree);
        struct rb_node *rb_next(struct rb_node *node);
        struct rb_node *rb_prev(struct rb_node *node);
```

A call to `rb_first()` will return a pointer to the first entry in the tree, while `rb_last()` returns the final entry. Moving forward and backward through the tree is a simple matter of calling `rb_next()` and `rb_prev()`. In all of these cases, a return value of `NULL` indicates that the requested node does not exist. 

Since the `rb_node` structures are embedded within some other structure of interest, finding the `rb_node` is a simple matter of using the right structure field. A call to one of the above functions will return a pointer to the embedded `rb_node` structure, however, and not the containing structure which is, normally, what the programmer really wants. This is the situation that the `container_of()` macro was created for, but, in this case, there is no need to use `container_of()` directly. Instead, use `rb_entry()`: 

```
rb_entry(pointer, type, member);
```

Where `pointer` is a pointer to an `rb_node` structure, `type` is the type of the containing structure, and `member` is the name of the `rb_node` structure within the container. 

Searching an existing tree for a value is simply a matter of starting at the root, then, for each node, comparing the value of that node against the target and following the left or right branch as necessary. So all rbtree search code tends to look like the following: 

```
struct my_stuff *my_rb_search(struct rb_root *root, int value)
        {
            struct rb_node *node = root->rb_node;  /* top of the tree */
    
            while (node)
    	{
    	    struct my_stuff *stuff = rb_entry(node, struct my_stuff, node);
    
    	    if (stuff->coolness > value)
    		node = node->rb_left;
    	    else if (stuff->coolness < value)
    		node = node->rb_right;
    	    else
    		return stuff;  /* Found it */
      	}
    	return NULL;
        }
```

Here, we are searching for a `struct my_stuff` whose `coolness` field matches the given `value`. An integer value is used for simplicity, but not all uses need be so simple. If the `coolness` of the root node is greater than the target value, then that value must be found in the left branch of the tree (if it is in the tree at all), so the search follows the `rb_left` branch and starts over. A search value greater than the current node's value indicates that the right branch should be used instead. Eventually this function will either find an exact match, or hit the bottom of the tree. 

The insertion case is a little trickier. The code must traverse the tree until it finds the leaf node where the insertion should take place. Once it has found that spot, the new node is inserted as a "red" node, and the tree is rebalanced if need be. Insertion code tends to have this form: 

```
void my_rb_insert(struct rb_root *root, struct my_stuff *new)
        {
            struct rb_node **link = &root->rb_node, *parent;
    	int value = new->coolness;
    
    	/* Go to the bottom of the tree */
    	while (*link)
    	{
    	    parent = *link;
    	    struct my_stuff *stuff = rb_entry(parent, struct my_stuff, parent);
    
    	    if (stuff->coolness > value)
    		link = &(*link)->rb_left;
    	    else
    		link = &(*link)->rb_right;
    	}
    
    	/* Put the new node there */
    	rb_link_node(new, parent, link);
    	rb_insert_color(new, root);
        }
```

In this case, the traversal of the tree looks similar to the search case. However, the `link` pointer is doubly indirected; in the end, it will be used to tell the rbtree code which branch pointer (`rb_left` or `rb_right`) should be set to point to the new entry. The code follows the tree all the way to the bottom, at which point the `parent` pointer identifies the parent of the new node, and `link` points to the appropriate field within `parent`. Then, a call is made to: 

```
void rb_link_node(struct rb_node *new_node,
                          struct rb_node *parent,
    		      struct rb_node **link);
```

This call will link the new node into the tree as a red node. After this call, however, the tree may no longer meet all the requirements for a red-black tree, and may thus need to be rebalanced. That work is done by calling: 

```
void rb_insert_color(struct rb_node *new_node, struct rb_root *tree);
```

Once that step is complete, the tree will be in consistent form. 

There is an important assumption built into the above example: the new value being inserted into the tree is not already present there. If that assumption is not warranted, a corrupted tree could result. If the possibility of a duplicated insertion exists, the code must be careful to test for an exact match (as is done in the search case) and stop (without inserting the node) if that match is found. 

Removal of a node from a tree is simpler; simply call: 

```
void rb_erase(struct rb_node *victim, struct rb_root *tree);
```

After the call, `victim` will no longer be part of `tree`, which may have been rebalanced as part of the operation. If one tree entry is being replaced by another with the same value, however, there is no need to go through the removal and insertion process. Instead, use: 

```
void rb_replace_node(struct rb_node *old, 
                             struct rb_node *new,
    			 struct rb_root *tree);
```

This call will quickly remove `old` from the tree, substituting `new` in its place. If `new` does not have the same value as `old`, however, the tree will be corrupted.
