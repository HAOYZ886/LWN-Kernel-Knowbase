---
title: A better DMA memory allocator
url: https://lwn.net/Articles/272693/
date: "March 10, 2008"
category: Direct memory access; swiotlb
author: "By Jonathan Corbet March 10, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 10, 2008

As any device driver author knows, hardware can be a pain sometimes. In the early days of Linux, peripherals attached to the ISA bus inflicted their particular variety of pain by being unable to use more than 24 bits to access memory. What that meant, in practical terms, was that ISA devices could not perform DMA operations on memory above 16MB. The PCI bus lifted that restriction, but, for some time, there were quite a few "PCI" devices that were minimally modified ISA peripherals; many of those retained the 16MB limit. 

To handle the needs of these devices, Linux has long maintained the DMA memory zone. Drivers which need to allocate memory from that zone would specify GFP_DMA in their allocation requests. The memory management code takes special care to keep memory in that zone available so that DMA requests can be satisfied. In this way, the system can provide reasonable assurance that memory will be available to perform DMA in ways which meet the special needs of this particularly challenged hardware. 

The only problem is that there aren't a whole lot of devices out there which still have the old 24-bit addressing limitation. So the DMA zone tends to sit idle. Meanwhile, there _are_ devices with other sorts of limitations. Many peripherals only handle 32-bit addresses, so their DMA buffers must be allocated in the bottom 4GB of memory. There is a subset, however, with stranger limitations - 30 or 31-bit addresses, for example. The kernel's DMA library provides a way for drivers to disclose that sort of embarrassing limitation, but the memory management code does not really help the DMA layer make allocations which satisfy those constraints. So drivers for such devices must use the DMA zone (which may not be present on all architectures), or hope that normal zone memory fits the bill. 

Andi Kleen has set out to clean up this situation with [a new DMA memory allocator](<http://lwn.net/Articles/272586/>). His solution is to take a chunk of memory out of the kernel's buddy allocator entirely and manage it in an entirely different way, forming a reserve pool for DMA allocations. The result is a bit of a departure from normal Linux memory management algorithms, but it may well be better suited to the task at hand. 

The new "mask" allocator grabs a configurable chunk of low memory at boot time. Allocations from this region are made with a separate set of calls, with the core API being: 

```
struct page *alloc_pages_mask(gfp_t gfp, unsigned size, u64 mask);
        void __free_pages_mask(struct page *page, unsigned size);
    
        void *get_pages_mask(gfp_t gfp, unsigned size, u64 mask);
        void free_pages_mask(void *mem, unsigned size);
```

`alloc_pages_mask()` looks a lot like the longstanding `alloc_pages()` function, but there's some important differences. The `size` parameter is the desired size of the allocation, rather than the "order" value used by `alloc_pages()`, and `mask` describes the range of usable addresses for this allocation. Though `mask` looks like a bitmask, it is really better understood as the address value that the allocated memory should have; "holes" in the mask would make no sense. 

A call to `alloc_pages_mask()` will first attempt to allocate the requested memory using the normal Linux memory allocator, on the assumption that the reserved DMA memory is an especially limited resource. If the allocation fails, perhaps because there's no physically-contiguous chunk of sufficient size available, then the allocator will dip into the reserved DMA pool. If the normal allocation succeeds, though, the allocated memory must still be tested against the maximum allowable address: the normal memory allocator, remember, has no support for allocating below an arbitrary address. So if the returned memory is out of bounds, it must be immediately freed and the reserved pool will be used instead. 

That reserved pool is not managed like the rest of memory. Rather than the buddy lists maintained by the slab allocator, the DMA allocator has a simple bitmap describing which pages are available. It will normally cycle through the entire memory region, allocating the next available chunk of sufficient size. If that chunk is above the memory limit, though, the allocator will move back to the lower end of the reserved pool and allocate from there instead. Since DMA allocations tend to be short-lived, one would expect that a suitable block of memory would either be available or become available in the near future. 

One other difference of note is that, unlike the slab allocator, the DMA allocator does not round memory allocation sizes up to the next power of two. DMA allocations can be relatively large, so that rounding can result in significant internal fragmentation and memory waste. 

At the next level up, Andi has added a new form of mempool which uses the DMA allocator: 

```
mempool_t *mempool_create_pool_pmask(int min_nr, int size, u64 mask);
```

This pool will behave like normal mempools, with the exception that all allocations will be below the limit passed in as `mask`. These pools are used in the block layer, where memory allocations for DMA must succeed. 

One might object that reserving a big chunk of low memory for this purpose reduces the total amount of memory available to the system - especially if the DMA allocator is cherry-picking normal memory whenever it can anyway. But the cost is not as bad as one might think. These patches do away with the old DMA zone, which, for all practical purposes, was already managed as a reserved (and often unused) memory area. Some 64-bit architectures also set aside a significant chunk (around 64MB) of low memory for the swiotlb - essentially a set of bounce buffers used for impedance matching between high memory (>4GB) buffers and devices which cannot handle more than 32-bit addresses. With Andi's patch set, the swiotlb, too, makes allocations from the DMA area and no longer has its own dedicated memory pool. So the total amount of memory set aside for I/O will not change very much; it could, in fact, get smaller. 

For most driver authors, there will be little in the way of required changes if this patch set gets merged. The DMA layer already allows drivers to specify an address mask with `dma_set_mask()`; with the DMA allocator in place, that mask will be better observed. The one change which might affect a few drivers is further down the line: eventually the `GFP_DMA` memory allocation flag will go away. Any driver which still uses this flag should set a proper mask instead. 

So far, there has been little discussion resulting from the posting of these patches. Silence does not mean assent, of course, but it would appear that there is little opposition to this set of changes.
