---
title: Pulling slabs out of struct page
url: https://lwn.net/Articles/871982/
date: "October 8, 2021"
category: "Memory management-Memory descriptors"
author: "By Jonathan Corbet October 8, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 8, 2021

For the time being, the effort to add [the folio concept](<https://lwn.net/Articles/849538/>) to the memory-management subsystem [appears to be stalled](<https://lwn.net/Articles/868598/>), but appearances can be deceiving. The numerous folio discussions have produced a number of points of consensus, though; one of those is that far too much of the kernel has to work with `page` structures to get its job done. As an example of how a subsystem might be weaned off of `struct page` usage, Matthew Wilcox has [split out the slab allocators](<https://lwn.net/ml/linux-mm/20211004134650.4031813-1-willy@infradead.org/>) in a 62-part patch set. The result may be a foreshadowing of changes to come in the memory-management subsystem. 

The kernel maintains one `page` structure for every physical page of memory that it manages. On a typical system with a 4KB page size, that means managing millions of those structures. A `page` structure tells the kernel about the state of the page it refers to: how it is being used, how many references to it exist, its position in various queues, and more. The required information varies depending on how any given page is being used at the moment; to accommodate this, `struct page` is a complicated mess of structures and unions. The [current definition of `struct page`](<https://elixir.bootlin.com/linux/v5.15-rc4/source/include/linux/mm_types.h#L31>) makes for good pre-Halloween reading, but those who truly want a good scare may want to see [what it looked like](<https://elixir.bootlin.com/linux/v4.17.19/source/include/linux/mm_types.h#L31>) before Wilcox cleaned things up for 4.18. 

One of the users of `struct page` is the set of slab allocators, which obtain blocks of pages ("slabs") from the kernel and hand them out in smaller, fixed-size chunks. They are used heavily, and their performance will affect the performance of the system as a whole, so it is not surprising that they reach into the memory-management subsystem at the lowest levels. To support this usage, many of the fields in `struct page` are there specifically for the slab allocators. Just to complicate things, the kernel has three slab allocators: SLAB (the original allocator, often used by Android), SLUB (often used for desktop and data-center systems), and SLOB (a tiny allocator intended for embedded systems). Each has its own needs for fields in `struct page`. 

Wilcox's patch set [creates a new `struct slab`](<https://lwn.net/ml/linux-mm/20211004134650.4031813-4-willy@infradead.org/>) by removing the relevant fields from `struct page`. The new structure is, in its entirety: 

```
struct slab {
    	unsigned long flags;
    	union {
    	    struct list_head slab_list;
    	    struct {	/* Partial pages */
    	        struct slab *next;
        #ifdef CONFIG_64BIT
    	        int slabs;	/* Nr of slabs left */
    	        int pobjects;	/* Approximate count */
        #else
    	        short int slabs;
    	        short int pobjects;
        #endif
    	    };
    	    struct rcu_head rcu_head;
    	};
    	struct kmem_cache *slab_cache; /* not slob */
    	/* Double-word boundary */
    	void *freelist;		/* first free object */
    	union {
    	    void *s_mem;	/* slab: first object */
    	    unsigned long counters;	/* SLUB */
    	    struct {			/* SLUB */
    	        unsigned inuse:16;
    	        unsigned objects:15;
    	        unsigned frozen:1;
    	    };
    	};
    
    	union {
    	    unsigned int active;	/* SLAB */
    	    int units;			/* SLOB */
    	};
    	atomic_t _refcount;
        #ifdef CONFIG_MEMCG
    	unsigned long memcg_data;
        #endif
        };
```

As can be seen, this structure still relies heavily on unions to overlay the information that each allocator needs to store with the page. Those could be eliminated by splitting the structure into three allocator-specific variants, but that would add complication to a patch set that is already large (and set to grow). 

It is worth noting that `struct slab` is really `struct page` in disguise; instances of `struct slab` overlay the corresponding `page` structure in the kernel's memory map. It is, essentially, the kernel's view of `struct page` for pages that are owned by a slab allocator, extricated from its coexistence with all of the other views of that structure. This means that `struct slab` must be laid out with care; some fields (`_refcount`, for example) are shared with `struct page`, and the results of a disagreement over its location would be unfortunate. To ensure that no such calamity occurs, Wilcox has included a set of compile-time tests verifying the offsets of the shared fields. 

After that, the remaining patches in the series convert various code in the slab allocators (and beyond) to use the new type. The SLUB conversion is done meticulously, in over 40 small steps. Wilcox described the conversion of the other allocators as ""slapdash"", done in a single patch each. Presumably a later version of the patch set will turn these proof-of-concept patches into a proper series of their own, but it's not entirely clear who will do that; Wilcox wrote in the cover letter: 

> I don't know the slab allocators terribly well, so I would be very grateful if one of the slab maintainers took over this effort. This is kind of a distraction from what I'm really trying to accomplish with folios, although it has found one minor bug. 

As of this writing, no slab maintainers (who tend to be thin on the ground in the best of times) have responded to this request. 

This might seem like a lot of work to put an old structure into a new form, but there are a number of reasons to want something like this. Just pulling the slab-specific fields out of `struct page` simplifies that structure significantly. Using a separate type makes it clear which variant of the `page` structure the code expects to deal with, and it adds a degree of type safety; it is no longer possible to accidentally access the wrong union fields. 

But the biggest benefit comes simply from beginning to separate the slab allocators from `struct page`. Eventually it may become possible to disassociate `struct slab` from `struct page` entirely and allocate it dynamically. That would be one small step toward encapsulating `struct page` within the core memory-management code and hiding it from the rest of the kernel, a change that would ease the much-needed task of improving page-level memory management. 

First, though, some variant of this work must make it into the mainline kernel. It should be an easier process than getting folios merged, but getting big changes into the memory-management code is never easy. The relative silence that has greeted this work so far might be a bit worrisome, especially since Wilcox has requested help, but it is the early days for this series still. Regardless of how `struct slab` fares, though, it provides an indication of the direction that the memory-management developers are trying to go.
