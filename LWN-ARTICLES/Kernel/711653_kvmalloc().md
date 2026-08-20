---
title: kvmalloc()
url: https://lwn.net/Articles/711653/
date: "January 17, 2017"
category: Memory management
author: "By Jonathan Corbet January 17, 2017"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 17, 2017

Patterns of code that repeat throughout the kernel are often a sign that a helper function may be appropriate. That is doubly true if those repeated patterns include repeated errors or other suboptimal techniques. The case of `kvmalloc()` would appear to be just one of those cases; it is the subject of [a recent patch set](<https://lwn.net/Articles/711452/>) from Michal Hocko that significantly cleans up a common memory-allocation pattern in the kernel. 

The kernel offers two fundamental mechanisms for allocating memory, both of which are, in turn, built on top of the kernel's page allocator. One of them, the slab allocator, will obtain physically contiguous memory in the kernel's own address space; this allocator is typically accessed via `kmalloc()`, though there are a number of other entry points as well. The alternative to the slab allocator is `vmalloc()`, which will return memory in a separate address space; that memory will be virtually contiguous but may be physically scattered. 

As a general rule, slab allocations are preferred for all but the largest of allocations. In the absence of memory pressure, the slab allocator will be faster, since it does not need to make address-space changes. The slab allocator works best with allocations that are less than one physical page (usually 4KB) in size, though. When memory gets fragmented, groups of physically contiguous pages can get hard to find, and system performance can suffer as the allocator struggles to create such groups. 

Allocations with `vmalloc()` do not need physically contiguous pages and are thus much more likely to succeed when memory is tight. But excessive use of `vmalloc()` is discouraged due to the extra overhead involved; each allocation done with `vmalloc()` requires page-table changes and a translation lookaside buffer invalidation. `vmalloc()` can only allocate entire pages, so it is not suitable for small requests. The address range available for `vmalloc()` allocations is also limited on 32-bit systems, which has historically been another disincentive to use this interface; that limitation is not present on 64-bit systems, though. 

There are a number of places in the kernel where a large allocation must be physically contiguous, but there are probably even more where that doesn't matter. In the latter case, the code doesn't have a reason to care which allocation method was used to obtain its memory, as long as the memory is available. For this sort of indifferent code, it can make sense to try an allocation first with the slab allocator, then fall back to `vmalloc()` should that attempt fail. And, indeed, the kernel is full of code fragments that do exactly that. 

However, as Hocko points out in the introduction to the `kvmalloc()` patch set, some of those fragments are ""really creative"" and many of them do not work as intended. For example, consider the following simple attempt at a fallback: 

```
memory = kmalloc(allocation_size, GFP_KERNEL);
        if (!memory)
            memory = vmalloc(allocation_size);
```

The problem here is that, for relatively small allocations (eight pages or less), `kmalloc()` [will retry indefinitely](<https://lwn.net/Articles/627419/>) rather than return failure. In such cases, the fallback path using `vmalloc()` will never be executed. Perhaps worse, the above `kmalloc()` call will go far out of its way in its attempts to satisfy the request. That could, for example, involve unchaining the dreaded out-of-memory killer to wreak havoc among unsuspecting processes. There are times when such drastic actions are needed, but memory-allocation code that has an explicit fallback path does not generally comprise one of those times. 

What is needed, of course, is a simple helper that implements an allocation using this fallback technique while taking care to minimize the amount of unnecessary collateral damage. To that end, Hocko's patch set introduces a few new functions: 

```
void *kvmalloc(size_t size, gfp_t flags);
        void *kvzalloc(size_t size, gfp_t flags);
        void *kvmalloc_node(size_t size, gfp_t flags, int node);
        void *kvzalloc_node(size_t size, gfp_t flags, int node);
```

As one might expect, `kvmalloc()` attempts to allocate `size` bytes from the slab allocator; it makes a point of using the `__GFP_NOWARN` and `__GFP_NORETRY` flags to minimize the cost (and avoid out-of-memory killer invocations) when the memory is not immediately available. If the attempt fails, `kvmalloc()` will fall back to `vmalloc()` to perform the allocation. The `kvzalloc()` variant will zero the memory before returning it. The `_node` versions request that the memory be allocated local to the given NUMA `node`. As with any other allocation function, these functions can still fail. 

Note that it doesn't generally make sense to use any of these functions with allocation requests that are smaller than a single page. Memory from `vmalloc()` is not available in sub-page granularity, so the fallback to `vmalloc()` will not be done in such cases. They also will not work as desired if called from atomic context, since `vmalloc()` cannot be used then. 

It is interesting to note that this is not the first attempt to add `kvmalloc()` to the kernel; [a version](<https://lwn.net/Articles/386535/>) was posted by Changli Gao in 2010. That version did not take the same level of care to avoid unfortunate side effects, though, and was never merged. Hocko's patch set, which also converts a large number of open-coded fallback implementations to the new functions, seems more likely to find its way into the mainline.
