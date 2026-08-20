---
title: Rationalizing scatter/gather chains
url: https://lwn.net/Articles/263343/
date: "December 28, 2007"
category: Direct memory access; Scattergather chaining
author: "By Jonathan Corbet December 28, 2007"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 28, 2007

The [chained scatterlist API](<http://lwn.net/Articles/256368/>) was arguably the most disruptive addition to 2.6.24, despite being a relatively small amount of code. This API allows kernel code to chain together scatter/gather lists for DMA I/O operations, resulting in a much larger maximum size for those operations. That, in turn, leads to better performance, especially in the block I/O subsystem. The idea of scatterlist chaining is generally popular, but there have been some complaints about the current implementation. As things stand, any code wanting to work with chained scatterlists must construct the chains itself \- an error-prone operation. So there is interest in making things better. 

One approach to improving the situation is the [sg_ring API](<http://lwn.net/Articles/263203/>), proposed by Rusty Russell. This patch does away with the current chaining approach; there are no more scatterlist entries which are actually chain pointers in disguise. Instead, Rusty introduces `struct sg_ring`: 

```
struct sg_ring
        {
    	struct list_head list;
    	unsigned int num, max;
    	struct scatterlist sg[0];
        };
```

The obvious change here is that the chaining has been moved out of the scatterlist itself and made into an explicit linked list. There are also variables tracking the current and maximum sizes of the list, which help reduce explicit housekeeping elsewhere. Some versions of the patch also add an integer `dma_num` field to hold the number of mapped scatter/gather entries, which can differ from the number initially set up by the driver. 

An `sg_ring` with a given number of scatterlist entries can be declared with this macro: 

```
DECLARE_SG_RING(name, max);
```

A ring should then be initialized with one of: 

```
void sg_ring_init(struct sg_ring *ring, unsigned int max);
        void sg_ring_single(struct sg_ring *ring, const void *buf,
                            unsigned int buflen);
```

The latter form is a shortcut for cases where a single-entry ring needs to be set up with a given buffer. 

Constructing a multi-entry ring is a matter of allocating as many separate `sg_ring` entries as needed and explicitly chaining them together using the `list` field. There is a helper macro for stepping through all of the entries in a ring while hiding the boundaries between the individual scatterlists: 

```
struct sg_ring *sg;
        int i;
    
        sg_ring_for_each(ring, sg, i) {
    	/* *sg is the scatterlist entry to operate on */
        }
```

Rusty has posted patches converting parts of the SCSI subsystem over to this API. As he points out, the conversion removes a fair amount of logic associated with the construction and destruction of large scatterlists. 

Jens Axboe, the creator of the chained scatterlist code, has [responded](<https://lwn.net/Articles/263345/>) that the current API was aimed at minimizing the effect on drivers for 2.6.24. It is not, he says, a finished product, and things are getting better. A look at [his git repository](<http://git.kernel.dk/?p=linux-2.6-block.git;a=shortlog;h=sg>) shows some API additions with a very similar goal to Rusty's work. 

Jens's work retains the current chaining mechanism, but wraps a structure and some helpers around it to make it easier to work with. So, in this view of the world, drivers will work with `struct sg_table`: 

```
struct sg_table {
    	struct scatterlist *sgl;        /* the list */
           	unsigned int nents;             /* number of mapped entries */
           	unsigned int orig_nents;        /* original size of list */
        };
```

An `sg_table` will be set up with: 

```
int sg_alloc_table(struct sg_table *table, unsigned int nents,
                           gfp_t gfp_flags);
```

This function does not allocate the `sg_table` structure, which must be passed in as a parameter. It does, however, allocate the memory to use for the actual `scatterlist` arrays and deal with the process of chaining them all together. So a driver needing to construct a large scatter/gather operation can now just do a single `sg_alloc_table()` call, then iterate through the list of scatterlist entries in the usual way. When the operation is complete, a call to 

```
void sg_free_table(struct sg_table *table);
```

will free the allocated memory. 

Sometime around the opening of the 2.6.25, a decision will have to be made on the direction of the chained scatterlist API. It may not be one of the most closely-watched kernel development events ever, but this decision will affect how high-performance I/O code is written in the future. As the author of the current chaining code, Jens probably starts with an advantage when it comes to getting his code merged. The nature of kernel development is such that nobody can ever be entirely sure, though; if a consensus builds that Rusty's approach is better, that is the way things will probably go. Stay tuned through the next merge window for the thrilling conclusion to this ongoing story.
