---
title: "Linux kernel design patterns - part 2"
url: https://lwn.net/Articles/336255/
date: "June 12, 2009"
category: "Development model-Patterns"
author: "June 12, 2009 This article was contributed by Neil Brown"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

June 12, 2009

This article was contributed by Neil Brown

[Last week](<http://lwn.net/Articles/336224/>) we discussed the value of enunciating kernel design patterns and looked at the design patterns surrounding reference counts. This week we will look at a very different aspect of coding and see why the kernel has special needs, and how those needs have been addressed by successful approaches. The topic under the microscope today is complex data structures. 

By "complex data structures" we mean objects that are composed of a variable number of simpler objects, possibly a homogeneous collection, possibly a heterogeneous collection. In general it is a structure to which objects can be added, from which objects can be removed, and in which objects can be found. The preferred way to work with such data structures when we study them in an introductory programming course is to use [Abstract Data Types](<http://en.wikipedia.org/wiki/Abstract_data_type>). 

#### Abstract Data Types

The idea behind Abstract Data Types is to encapsulate the entire implementation of a data structure, and provide just a well defined interface for manipulating it. The benefit of this approach is that it provides a clean separation. The data type can be implemented with no knowledge of the application which might end up using it, and the application can be implemented with no knowledge of the implementation details of the data type. Both sides simply write code based on the interface which works like a contract to explicitly state what is required and what can be expected. 

On the other hand, one of the costs of this approach is tightly connected with the abstract nature of the interface. The point of abstracting an interface is to remove unnecessary details. This is good for introductory computing students, but is bad for kernel programmers. In kernel programming. performance is very important, coming as a close third after correctness and maintainability, and sometimes taking precedence over maintainability. Not all code paths in the kernel are performance-critical, but many are, and the development process benefits from the same data structures being using in both performance critical and less critical paths. So it is essential that data types are not overly abstracted, but that all details of the implementation are visible so that the programmer can make optimal choices when using them. 

So the first principle of data structures in the kernel is not to hide detail. To see how this applies, and to discover further principles from which to extract patterns, we will explore a few of the more successful data types used in the Linux kernel. 

#### Linked Lists

Starting simply, the first data type we will explore are doubly linked lists. These are implemented by a single include file, `<linux/list.h>`. There is no separate ".c" file with any library of support routines. All of the code for handling linked lists is simple enough to be implemented using inline functions. Thus it is very easy to use this implementation in any other (GPLv2-licensed) project. 

There are two aspects of the "list.h" lists which are worth noting as they point to possible patterns. The first is `struct list_head`, which serves not only as the head of a list, but also as the anchor in items that are on a list. Your author has seen other linked list implementations which required that the first and second element in any data structures meant to be stored in lists be the "next" and "previous" pointers, so that common list-walking code could be used on a variety of different data structures. Linux kernel lists do not suffer from this restriction. The `list_head` structure can be embedded anywhere in a data structure, and the `list_head`s from a number of instances of that structure can be linked together. The containing structure can be found from a `->next` or `->prev` pointer using the `list_entry()` macro. 

There are at least two benefits of this approach. One is that the programmer still has full control of placement of fields in the structure in case they need to put important fields close together to improve cache utilization. The other is that a structure can easily be on two or more lists quite independently, simply by having multiple `struct list_head` fields. 

This practice of embedding one structure inside another and using `container_of()` (which is the general form of `list_entry()`) to get the parent from the child structure is quite common in the Linux kernel and is somewhat reminiscent of object oriented programming. The container is like a subtype of the embedded structure. 

The other noteworthy aspect of list.h is the proliferation of "for_each" macros - the macros that make it easy to walk along a list looking at each item in turn. There are 20 of them (and that isn't counting the four more in `rculist.h` which I'll choose to ignore in the hope of brevity). 

There are a few different reasons for this. The simplest are that 

  * We sometimes want to walk the list in the "reverse" direction (following the "prev" link). There are five macros that go backward, and 15 that go forward. 

  * We sometimes want to start in the middle of a list and "continue" on from there, so we have four "continue" macros and three "from" macros which interpret that starting point slightly differently. 

  * We sometimes want to work with the `struct list_head` embedded in the target structure, but often we really want to use the `list_entry()` macro to get the enclosing structure; we find it easiest if the "for_each" macro does that for us. This provides the "entry" versions of the "for_each" macro, of which there are 13 (more than half). 

Getting to the more subtle reasons, we sometimes want to be able to delete the "current" item without upsetting the walk through the list. This requires that a copy of the "next" pointer be taken before providing "this" entry to be acted upon, thus yielding the eight "safe" macros. An "ADT" style implementation of linked lists would quite likely only provide "safe" versions so as to hide these details. However kernel programmers don't want to waste the storage or effort for that extra step in the common case were it isn't needed. 

Then there is the fact that we actually have two subtly different types of linked lists. Regular linked lists use `struct list_head` as the head of the list. This structure contains a pointer to the start and to the end. In some use cases, finding the end of the list is not needed, and being able to halve the size of the head of the list is very valuable. One typical use case of that kind is a hash table where all these heads need to go in an array. To meet this need, we have the `hlist`, which is very similar to the regular list, except that only one pointer is needed in `struct hlist_head`. This accounts for six of the different "for_each" macros. 

If we had every possibly combination of forward or reverse, continue or not, entry or not, safe or not, and regular or hlist, we would have 32 different macros. In fact, only 19 of these appear to have been needed and, thus, coded. We certainly could code the remaining eleven, but as having code that is never used tends to be frowned upon, it hasn't happened. 

The observant reader will have noticed a small discrepancy in some of the above numbers. Of the 20 macros, there is one that doesn't fit the above patterns, and it drives home the point that was made earlier about kernel programmers valuing performance. This final "for_each" macro is `__list_for_each()`. All of the other macros use the "prefetch" function to suggest that the CPU starts fetching the `->next` pointer at the start of each iteration so that it will already be available in cache when the next iteration starts (though the "safe" macros actually fetch it rather than prefetch it). While this will normally improve performance, there are cases when it will slow things down. When the walk of the list will almost always abort very early - usually only considering the first item - the prefetch will often be wasted effort. In these cases (currently all in the networking code) the `__list_for_each()` macro is available. It does not prefetch anything. Thus people having very strict performance goals can have a better chance of getting the performance they want. 

So from this simple data structure we can see two valuable patterns that are worth following. 

  * **Embedded Anchor** : A good way to include generic objects in a data structure is to embed an anchor in them and build the data structure around the anchors. The object can be found from the anchor using `container_of()`. 

  * **Broad Interfaces** : Don't fall for the trap of thinking that "one size fits all". While having 20 or more macros that all do much the same thing is uncommon, it can be a very appropriate way of dealing with the complexity of finding the optimal solution. Trying to squeeze all possibilities into one narrow interface can be inefficient and choosing not to provide for all possibilities is counter-productive. Having all the permutations available encourages developers to use the right tool for the job and not to compromise. In 2.6.30-rc4, there are nearly 3000 uses of `list_for_each_entry()`, about 1000 of `list_for_each_entry_safe()`, nearly 500 of `list_for_each()`, and less than 1000 of all the rest put together. The fact that some are used rarely in no way reduces their importance. 

#### RB-trees

Our next data structure is the RB-Tree or [red-black tree](<http://lwn.net/Articles/184495/>). This is a semi-balanced, binary search tree that generally provides order "log(n)" search, insert, and delete operations. It is implemented in `<linux/rbtree.h>` and `lib/rbtree.c`. It has strong similarities to the `list.h` lists in that it embeds an anchor (`struct rb_node`) in each data structure and builds the tree from those. 

The interesting thing to note about rbtree is that there is no search function. Searching an rbtree is really a very simple operation and can be implemented in just a few lines as shown by the examples at the top of `rbtree.h`. A search function certainly could be written, but the developer chose not to. The main reason, which should come as no surprise, is performance. To write a search function, you need to pass the "compare" function into that search function. To do that in C, you would need to pass a function pointer. As compare functions are often very simple, the cost of following the function pointer and making the function call would often swamp the cost of doing the comparison itself. It turns out that having the whole search operation compiled as one function makes for more efficient code. The same performance could possibly be achieved using inline functions or macros, but given that the search function itself is so short, it hardly seems worthwhile. 

Note also that rbtree doesn't exactly provide an insert function either. Rather, the developer needs to code a search; if the search fails, the new node must be inserted at the point where it was found not to exist and the tree must be rebalanced. There are functions for this final insertion and rebalancing as they are certainly complex enough to deserve separate functions. 

By giving the developer the responsibility for search and for some of insert, the rbtree library actually is giving a lot of valuable freedom. The pattern of "search for an entry but if it isn't there, insert one" is fairly common. However the details of what happens between the "search" and "add" phases is not always the same and so not something that can easily be encoded in a library. By providing the basic tools and leaving the details up to the specific situation, users of rbtree find themselves liberated, rather than finding themselves fighting with a library that almost-but-not-quite does what they want. 

So this example of rbtrees re-enforces the "embedded anchors" pattern and suggests a pattern that providing tools is sometimes much more useful than providing complete solutions. In this case, the base data structures and the tools required for insert, remove, and rebalance are provided, but the complete solution can still be tailored to suit each case. 

This pattern also describes the kernel's approach to hash tables. These are a very common data structure, but there is nothing that looks even vaguely like a definitive implementation. Rather the basic building blocks of the hlist and the array are available along with some generic functions for calculating a hash (`<linux/hash.h>`). Connecting these to fit a given purpose is up to the developer. 

So we have another pattern: 

  * **Tool Box** : Sometimes it is best not to provide a complete solution for a generic service, but rather to provide a suite of tools that can be used to build custom solutions. 

#### Radix tree

Our last data structure is the [Radix tree](<http://lwn.net/Articles/175432/>). The Linux kernel actually has two radix tree implementations. One is in `<linux/idr.h>` and `lib/idr.c`, the other in `<linux/radix-tree.h>` and `lib/radix-tree.c`. Both provide a mapping from a number (`unsigned long`) to an arbitrary pointer (`void *`), though radix-tree also allows up to two "tags" to be stored with each entry. For the purposes of this article we will only be looking at one of the implementations (the one your author is most familiar with) - the radix-tree implementation. 

Radix-tree follows the pattern we saw in `list.h` of having multiple interfaces rather than trying to pack lots of different needs into the one interface. `list.h` has 20 "for_each" macros; radix-tree has six "lookup" functions, depending on whether we want just one item or a range (gang lookups), or whether we want to use the tags to restrict the search (tag lookups) and whether we want to find the place where the pointer is stored, rather than the pointer that is stored there (this is needed for the subtle locking strategies of the page cache). 

However radix-tree does not follow the embedded anchor pattern of the earlier data structures, and that is why it is interesting. For lists and rbtree, the storage needed for managing the data structure is exactly proportional to the number of items in the data structure on a one-to-one basis, so keeping this storage in those item works perfectly. For a radix-tree, the storage needed is a number of little arrays, each of which refers to a number of items. So embedding these arrays, one each, in the items cannot work. This means that, unlike list.h and rbtree, radix-tree will sometimes need to allocate some memory in order to be able to add items to the data structure. This has some interesting consequences. 

In the previous data structures (lists and rbtrees), we made no mention of locking. If locking is needed, then the user of the data structure is likely to know the specific needs so all locking details are left up to the caller (we call that "caller locking" as opposed to "callee locking". Caller locking is more common and generally preferred). This is fine for lists and rbtrees as nothing that they do internally is affected particularly by locking. 

This is not the case if memory allocation is needed, though. If a process needs to allocate memory, it is possible that it will need to sleep while the memory management subsystem writes data out to storage to make memory available. There are various locks (such as spinlocks) that may not be held while a process sleeps. So there is the possibility for significant interaction between the need to allocate memory internally to the radix-tree code, and the need to hold locks outside the radix-tree code. 

The obvious solution to this problem (once the problem is understood) is to preallocate the maximum amount of memory needed before taking the lock. This is implemented within radix-tree with the `radix_tree_preload()` function. It manages a per-CPU pool of available `radix_tree` nodes and makes sure the pool is full and will not be used by any other radix-tree operation. Thus, bracketing `radix_tree_insert()` with calls to `radix_tree_preload()` and `radix_tree_preload_end()` ensures that the `radix_tree_insert()` call will not fail due to lack of memory (though the `radix_tree_preload()` might) and that there will be no unpleasant interactions with locking. 

#### Summary

So we can now summarize our list of design patterns that we have found that work well with data structures (and elsewhere) in the kernel. Those that have already been detailed are briefly included in this list too for completeness. 

  * **Embedded Anchor**. This is very useful for lists, and can be generalized as can be seen if you explore kobjects (an exercise left to the reader). 

  * **Broad Interfaces**. This reminds us that trying to squeeze lots of use-cases in to one function call is not necessary - just provide lots of function calls (with helpful and (hopefully) consistent names). 

  * **Tool Box**. Sometimes it is best not to provide a complete solution for a generic service, but rather to provide a suite of tools that can be used to build custom solutions. 

  * **Caller Locks**. When there is any doubt, choose to have the caller take locks rather than the callee. This puts more control in that hands of the client of a function. 

  * **Preallocate Outside Locks**. This is in some ways fairly obvious. But it is very widely used within the kernel, so stating it explicitly is a good idea. 

[Next week](<https://lwn.net/Articles/336262/>) we will complete our investigation of kernel design patterns by taking a higher level view and look at some patterns relating to the design of whole subsystems. 

#### Exercises

For those who would like to explore these ideas further, here are some starting points. 

  1. Make a list of all data structures that are embedded, by exploring all uses of the "container_of" macro. Of these, make a list of pairs that are both embedded in the same structure (modeling multiple inheritance). Comment on how this reflects on the general usefulness of multiple inheritance. 

  2. Write a implementation of [skiplists](<http://en.wikipedia.org/wiki/Skiplist>) that would be suitable for in-kernel use. Consider the applicability of each of the patterns discussed in this article. Extra credit for leveraging list.h lists. 

  3. Linux contains a mempool library which radix-tree chooses not to use, preferring it's own simple pool (in radix_tree_preload). Examine the consequences of changing radix-tree to use mempool, and of changing mempool to be usable by radix-tree. Provide patches to revert this design choice in radix-tree, or a pattern to explain this design choice. 

  4. Compare the radix-tree and idr implementations to see if one could be implemented using the other without sacrificing correctness, maintainability or performance. Provide either an explanation of why they should stay separate, or a patch to replace one with the other.
