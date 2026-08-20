---
title: Flexible arrays
url: https://lwn.net/Articles/345273/
date: "August 5, 2009"
category: Flexible arrays
author: "By Jonathan Corbet August 5, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 5, 2009

Kernel developers must keep in mind many constraints which are unique to that programming environment; one of those is that memory allocations become less reliable as they get larger. Single-page allocations will, for all practical purposes, always succeed. A request for two physically-contiguous pages has a high probability of working, but each doubling of the size decreases the chances of a successful allocation. The fragmentation of memory which occurs over the system's life time makes it increasingly hard to find groups of groups of physically-contiguous pages on demand. So large allocations are strongly discouraged. 

Kernel programmers will sometimes respond to this problem by allocating pages with `vmalloc()`. Memory allocated this way is virtually contiguous, but physically scattered. So, as long as physically-contiguous pages are not needed, `vmalloc()` looks like a good solution to the problem. It's not ideal, though. On 32-bit systems, memory from `vmalloc()` must be mapped into a relatively small address space; it's easy to run out. On SMP systems, the page table changes required by `vmalloc()` allocations can require expensive cross-processor interrupts on all CPUs. And, on all systems, use of space in the `vmalloc()` range increases pressure on the translation lookaside buffer (TLB), reducing the performance of the system. 

So it would be nice to have a mechanism which could handle the allocation of large arrays in a manner which (1) is reliable, and (2) does not use `vmalloc()`. To date, any such mechanisms have generally been pieced together by developers solving a specific problem; there has been nothing designed for more general use. That has changed, though, with the merging of the "flexible array" mechanism, written by Dave Hansen, for 2.6.31-rc5. 

A flexible array holds an arbitrary (within limits) number of fixed-sized objects, accessed via an integer index. Sparse arrays are handled reasonably well. Only single-page allocations are made, so memory allocation failures should be relatively rare. The down sides are that the arrays cannot be indexed directly, individual object size cannot exceed the system page size, and putting data into a flexible array requires a copy operation. It's also worth noting that flexible arrays do no internal locking at all; if concurrent access to an array is possible, then the caller must arrange for appropriate mutual exclusion. 

The creation of a flexible array is done with: 

```
#include <linux/flex_array.h>
    
        struct flex_array *flex_array_alloc(int element_size, int total, gfp_t flags);
```

The individual object size is provided by `element_size`, while `total` is the maximum number of objects which can be stored in the array. The `flags` argument is passed directly to the internal memory allocation calls. With the current code, using `flags` to ask for high memory is likely to lead to notably unpleasant side effects. 

Storing data into a flexible array is accomplished with a call to: 

```
int flex_array_put(struct flex_array *array, int element_nr, void *src, gfp_t flags);
```

This call will copy the data from `src` into the `array`, in the position indicated by `element_nr` (which must be less than the maximum specified when the array was created). If any memory allocations must be performed, `flags` will be used. The return value is zero on success, a negative error code otherwise. 

There might possibly be a need to store data into a flexible array while running in some sort of atomic context; in this situation, sleeping in the memory allocator would be a bad thing. That can be avoided by using `GFP_ATOMIC` for the `flags` value, but, often, there is a better way. The trick is to ensure that any needed memory allocations are done before entering atomic context, using: 

```
int flex_array_prealloc(struct flex_array *array, int start, int end, gfp_t flags);
```

This function will ensure that memory for the elements indexed in the range defined by `start` and `end` has been allocated. Thereafter, a `flex_array_put()` call on an element in that range is guaranteed not to block. 

Getting data back out of the array is done with: 

```
void *flex_array_get(struct flex_array *fa, int element_nr);
```

The return value is a pointer to the data element, or NULL if that particular element has never been allocated. 

Note that it is possible to get back a valid pointer for an element which has never been stored in the array. Memory for array elements is allocated one page at a time; a single allocation could provide memory for several adjacent elements. The flexible array code does not know if a specific element has been written to; it only knows if the associated memory is present. So a `flex_array_get()` call on an element which was never stored in the array has the potential to return a pointer to random data. If the caller does not have a separate way to know which elements were actually stored, it might be wise, at least, to add `GFP_ZERO` to the `flags` argument to ensure that all elements are zeroed. 

There is no way to remove a single element from the array. It is possible, though, to remove all elements with a call to: 

```
void flex_array_free_parts(struct flex_array *array);
```

This call frees all elements, but leaves the array itself in place. Freeing the entire array is done with: 

```
void flex_array_free(struct flex_array *array);
```

As of this writing, there are no users of flexible arrays in the mainline kernel. The functions described here are also not exported to modules; that will probably be fixed when somebody comes up with a need for it.
