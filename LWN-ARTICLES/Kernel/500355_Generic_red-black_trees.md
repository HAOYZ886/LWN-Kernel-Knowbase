---
title: "Generic red-black trees"
url: https://lwn.net/Articles/500355/
date: "June 5, 2012"
category: "Red-black trees"
author: "By Jonathan Corbet June 5, 2012"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 5, 2012

Red-black trees (or "rbtrees") are used throughout the kernel to maintain a sorted list of related items. For example, the block I/O subsystem uses an rbtree to maintain a list of outstanding requests sorted by sector number, and the scheduler has an rbtree of runnable tasks sorted by a quantity known as "virtual runtime." The rbtree interface was described in [this 2006 article](<https://lwn.net/Articles/184495/>); it has since been extended, but the core features of the API remain the same. In particular, users must provide their own functions for inserting nodes into the tree and performing searches; that allows the creation of highly-optimized rbtrees containing arbitrary types of structures. 

There is some appeal to being able to hand-code the search and insertion functions, but there would also be value in generic implementations. The amount of code in the kernel would shrink slightly and the task of debugging those functions would, hopefully, only have to be done once. So it is arguably surprising that nobody has proposed a generic rbtree implementation for all these years. Just as surprising is the fact that two independent generic implementations surfaced at about the same time. 

The simpler of the two can be found in [this patch set](<https://lwn.net/Articles/498895/>) from Kent Overstreet. In this version, rbtree users are required to provide a function to compare two rbtree nodes with this prototype: 

```
typedef int (rb_cmp_t) (struct rb_node *l, struct rb_node *r);
```

With that in place, the following new functions become available: 

```
int rb_insert(struct rb_root *root, struct rb_node *new, rb_cmp_t cmp);
        void rb_insert_allow_dup(struct rb_root *root, struct rb_node *new,
    			     rb_cmp_t cmp);
        struct rb_node *rb_search(struct rb_root *root, struct rb_node *search,
    			      rb_cmp_t cmp);
        struct rb_node *rb_greater(struct rb_root *root, struct rb_node *search,
    			       rb_cmp_t cmp);
```

As can be seen from the prototypes, all of these functions deal directly with `rb_node` structures. Only the comparison function needs to know about what sort of structure the `rb_node` is embedded within. There is no compile-time mechanism to ensure that the comparison function expects the actual structures found in the tree, but one assumes any errors along those lines will show themselves quickly at run time. 

One potential problem is that `rb_search()` and `rb_greater()` need to know what is being searched for; that, in turn, requires creating and passing in one of the structures stored in the tree. In some situations, that structure may need to be created on the stack, which is a clear problem if the structure is large. Unfortunately, in the block subsystem example mentioned above, that structure (`struct request`) is large indeed. Kent has tried to work around this problem by providing inlined versions (called `_rb_search()` and `_rb_greater()`) that, with luck, will cause the stack allocation to be optimized away. That depends on all supported versions of the compiler doing the right thing on all architectures, though, which may make some people nervous. 

The alternative patch set, [posted by Daniel Santos](<https://lwn.net/Articles/499924/>), is significantly more complicated. It can be thought of as a set of tricky preprocessor macros and inline functions that serve as a template for the creation of type-specific rbtree implementations. Here, too, one starts with the creation of a comparison function: 

```
typedef long (*rb_compare_f)(const void *a, const void *b);
```

In this case, the comparison function will be passed pointers to the actual key value stored in the rbtree node. One then defines an "rbtree interface" with this daunting macro: 

```
RB_DEFINE_INTERFACE(prefix, container_type, root_member,
    			left_pointer, right_pointer,
    			object_type, rbnode_member, key_member,
    			flags, comparison_function, augment_func);
```

Eleven arguments are a lot to keep track of, so it makes sense to discuss them in the context of an example (taken from Daniel's patch). The CPU scheduler defines a type (`struct cfs_rq` in `kernel/sched/sched.h`) to represent a run queue; each run queue contains a red-black tree called `tasks_timeline`. To optimize the location of the highest-priority task to run, the scheduler keeps a pointer to the leftmost tree node in `rb_leftmost`. The entries in the red-black tree are of type `struct sched_entity` (defined in `<linux/sched.h>`); the embedded `rb_node` structure is called `run_node`, and the key used to sort the tree is the `u64` value `vruntime`. 

To create the scheduler's tree using the generic mechanism, Daniel adds this declaration: 

```
RB_DEFINE_INTERFACE(
    	fair_tree,
    	struct cfs_rq, tasks_timeline, rb_leftmost, /* no rightmost */,
    	struct sched_entity, run_node, vruntime,
    	0, compare_vruntime, 0)
```

Here, `fair_tree` is the "prefix" value used to generate the names of the tree functions—see below. The next line describes the structure containing the tree (`struct cfs_rq`), the name of the tree, and the name of the leftmost pointer used by the scheduler; there is no rightmost pointer (nobody cares about the lowest-priority task), so that parameter is simply left blank. The line after that describes the structure contained within the tree (`struct sched_entity`), the name of its embedded `rb_node` structure, and the name of the key value. Finally, no flags are given, `compare_vruntime()` is the comparison function, and, since this is not an [augmented tree](<https://lwn.net/Articles/388118/>), there is no augmented callback function. Yes, it lacks a semicolon—the macro supplies that itself. 

The result is a new set of functions like: 

```
struct sched_entity *fair_tree_insert(struct cfs_rq runqueue, 
    					  struct sched_entity *entity);
        void fair_tree_remove(struct cfs_rq runqueue, struct sched_entity *entity);
        struct sched_entity *fair_tree_find(struct cfs_rq runqueue, u64 *key);
        /* ... */
```

These functions are all defined with the proper type, so the compiler will ensure that they are always called with the proper argument types. Everything is defined as `__always_inline`, so the implementations will be inlined at the place where they are called. That should eliminate any performance penalty caused by out-of-line implementations (as seen in Kent's patch), but, so far, nobody seems to have tried to measure that penalty. 

There has been little in the way of review of these patches in general. They represent different approaches to the task of creating a generic red-black implementation, one emphasizing simplicity while the other emphasizes explicitness and type safety. Which version might be inserted into the mainline kernel—if either goes in—is entirely unclear at this point.
