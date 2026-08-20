---
title: "The kernel's genpool subsystem"
url: https://lwn.net/Articles/729653/
date: "August 3, 2017"
category: Memory management
author: "By Jonathan Corbet August 3, 2017"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 3, 2017

The kernel is a huge program; among other things, that means that many problems encountered by a kernel developer have already been solved somewhere else in the tree. But those solutions are not always well known or documented. Recently, a seasoned developer [confessed](<https://lwn.net/Articles/729657/>) to having never encountered the "genpool" memory allocator. This little subsystem does not appear in the kernel documentation, and is likely to be unknown to others as well. In the interest of fixing both of those problems, here is an overview of genpool (or "genalloc") and what it does. 

There are a number of memory-allocation subsystems in the kernel, each aimed at a specific need. Sometimes, however, a kernel developer needs to implement a new allocator for a specific range of special-purpose memory; often that memory is located on a device somewhere. The author of the driver for that device can certainly write a little allocator to get the job done, but that is the way to fill the kernel with dozens of poorly tested allocators. Back in 2005, Jes Sorensen lifted one of those allocators from the sym53c8xx_2 driver and [posted it](<https://lwn.net/Articles/125842/>) as a generic module for the creation of _ad hoc_ memory allocators. This code was merged for the 2.6.13 release; it has been modified considerably since then. 

The action begins with the creation of a pool using one of: 

```
#include <linux/genalloc.h>
    
        struct gen_pool *gen_pool_create(int min_alloc_order, int nid);
        struct gen_pool *devm_gen_pool_create(struct device *dev, int min_alloc_order,
    				          int nid, const char *name);
```

A call to `gen_pool_create()` will create a pool. The granularity of allocations is set with `min_alloc_order`; it is a log-base-2 number like those used by the page allocator, but it refers to bytes rather than pages. So, if `min_alloc_order` is passed as `3`, then all allocations will be a multiple of eight bytes. Increasing `min_alloc_order` decreases the memory required to track the memory in the pool. The `nid` parameter specifies which NUMA node should be used for the allocation of the housekeeping structures; it can be `-1` if the caller doesn't care. 

The "managed" interface `devm_gen_pool_create()` ties the pool to a specific device. Among other things, it will automatically clean up the pool when the given device is destroyed. 

A pool is shut down with: 

```
void gen_pool_destroy(struct gen_pool *pool);
```

It's worth noting that, if there are still allocations outstanding from the given `pool`, this function will take the rather extreme step of invoking `BUG()`, crashing the entire system. You have been warned. 

A freshly created pool has no memory to allocate. It is fairly useless in that state, so one of the first orders of business is usually to add memory to the pool. That can be done with one of: 

```
int gen_pool_add(struct gen_pool *pool, unsigned long addr, size_t size, int nid);
        int gen_pool_add_virt(struct gen_pool *pool, unsigned long virt, phys_addr_t phys,
    		 	  size_t size, int nid);
```

A call to `gen_pool_add()` will place the `size` bytes of memory starting at `addr` (in the kernel's virtual address space) into the given `pool`, once again using `nid` as the node ID for ancillary memory allocations. The `gen_pool_add_virt()` variant associates an explicit physical address with the memory; this is only necessary if the pool will be used for DMA allocations. 

The functions for allocating memory from the pool (and putting it back) are: 

```
unsigned long gen_pool_alloc(struct gen_pool *pool, size_t size);
        void *gen_pool_dma_alloc(struct gen_pool *pool, size_t size, dma_addr_t *dma);
        extern void gen_pool_free(struct gen_pool *pool, unsigned long addr, size_t size);
```

As one would expect, `gen_pool_alloc()` will allocate `size` bytes from the given `pool`. The `gen_pool_dma_alloc()` variant allocates memory for use with DMA operations, returning the associated physical address in the space pointed to by `dma`. This will only work if the memory was added with `gen_pool_add_virt()`. Note that this function departs from the usual genpool pattern of using `unsigned long` values to represent kernel addresses; it returns a `void *` instead. 

That all seems relatively simple; indeed, some developers clearly found it to be too simple. After all, the interface above provides no control over how the allocation functions choose which specific piece of memory to return. If that sort of control is needed, the following functions will be of interest: 

```
unsigned long gen_pool_alloc_algo(struct gen_pool *pool, size_t size,
        				      genpool_algo_t algo, void *data);
        extern void gen_pool_set_algo(struct gen_pool *pool, genpool_algo_t algo,
    				  void *data);
```

Allocations with `gen_pool_alloc_algo()` specify an algorithm to be used to choose the memory to be allocated; the default algorithm can be set with `gen_pool_set_algo()`. The `data` value is passed to the algorithm; most ignore it, but it is occasionally needed. One can, naturally, write a special-purpose algorithm, but there is a fair set already available: 

  * `gen_pool_first_fit` is a simple first-fit allocator; this is the default algorithm if none other has been specified. 

  * `gen_pool_first_fit_align` forces the allocation to have a specific alignment (passed via `data` in a `genpool_data_align` structure). 

  * `gen_pool_first_fit_order_align` aligns the allocation to the order of the size. A 60-byte allocation will thus be 64-byte aligned, for example. 

  * `gen_pool_best_fit`, as one would expect, is a simple best-fit allocator. 

  * `gen_pool_fixed_alloc` allocates at a specific offset (passed in a `genpool_data_fixed` structure via the `data` parameter) within the pool. If the indicated memory is not available the allocation fails. 

There is a handful of other functions, mostly for purposes like querying the space available in the pool or iterating through chunks of memory. Most users, however, should not need much beyond what has been described above. With luck, wider awareness of this module will help to prevent the writing of special-purpose memory allocators in the future.
