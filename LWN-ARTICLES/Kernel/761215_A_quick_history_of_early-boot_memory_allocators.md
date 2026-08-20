---
title: "A quick history of early-boot memory allocators"
url: https://lwn.net/Articles/761215/
date: "July 30, 2018"
category: "Memory management-During early boot"
author: "July 30, 2018 This article was contributed by  Mike Rapoport"
---

July 30, 2018

This article was contributed by Mike Rapoport

One might think that memory allocation during system startup should not be difficult: almost all of memory is free, there is no concurrency, and there are no background tasks that will compete for memory. Even so, boot-time memory management is a tricky task. Physical memory is not necessarily contiguous, its extents change from system to system, and the detection of those extents may be not trivial. With NUMA things are even more complex because, in order to satisfy allocation locality, the exact memory topology must be determined. To cope with this, sophisticated mechanisms for memory management are required even during the earliest stages of the boot process. 

One could ask: "so why not use the same allocator that Linux uses normally from the very beginning?" The problem is that the primary Linux page allocator is a complex beast and it, too, needs to allocate memory to initialize itself. Moreover, the page-allocator data structures should be allocated in a NUMA-aware way. So another solution is required to get to the point where the memory-management subsystem can become fully operational. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

In the early days, Linux didn't have an early memory allocator; in the 1.0 kernel, memory initialization was not as robust and versatile as it is today. Every subsystem initialization call, or simply any function called from `start_kernel()`, had access to the starting address of the single block of free memory via the global `memory_start` variable. If a function needed to allocate memory it just increased `memory_start` by the desired amount. By the time v2.0 was released, Linux was already ported to five more architectures, but boot-time memory management remained as simple as in v1.0, with the only difference being that the extents of the physical memory were detected by the architecture-specific code. It should be noted, though, that hardware in those days was much simpler and memory configurations could be detected more easily. 

Up until version 2.3.23pre3, all early memory allocations used global variables indicating the beginning and end of free memory and adjusted them accordingly. Luckily, the page and slab allocators were available early, so heavy memory users, such as `buffers_init()` and `page_cache_init()`, could use them. Still, as hardware evolved and became more sophisticated, the architecture-specific code dealing with memory had grown quite a bit of complex cruft. 

The 2.3.23pre3 patch set included the first bootmem allocator implementation, which used a bitmap to represent the status of each physical memory page. Cleared bits identified available pages, while set bits meant that the corresponding memory pages were busy or absent. All the generic functions that tweaked `memory_start` and the i386 initialization code were converted to use bootmem, but other architectures were left behind. They were converted by the time version 2.3.48 was ready. Meanwhile, Linux was ported to Itanium (ia64), which was the first architecture to start off using bootmem. 

Over time, memory detection has evolved from simply asking the BIOS for the size of the extended memory block to dealing with complex tables, pieces, banks, and clusters. In particular, the Power64 architecture came prepared, bringing with it the [Logical Memory Block allocator](<https://lwn.net/Articles/387083/>) (or LMB). With LMB, memory is represented as two arrays of regions. The first array describes the physically contiguous memory areas available in the system, while the second array tracks allocated regions. The LMB allocator made its way into 32-bit PowerPC when the 32-bit and 64-bit architectures were merged. Later on it was adopted by SPARC. Eventually LMB made its way to other architectures and became what is now known as memblock. 

The memblock allocator provides two basic primitives that are used as the base for more complex allocation APIs: [`memblock_add()`](<https://elixir.bootlin.com/linux/latest/source/mm/memblock.c#L596>) for registering a physical memory range, and [`memblock_reserve()`](<https://elixir.bootlin.com/linux/latest/source/mm/memblock.c#L714>) to mark a range as busy. Both of these are based, in the end, on [`memblock_add_range()`](<https://elixir.bootlin.com/linux/latest/source/mm/memblock.c#L489>), which adds a range to either of the two arrays. 

The major drawback of bootmem is the bitmap initialization. To create this bitmap, it is necessary to know the physical memory configuration. What is the correct size of the bitmap? Which memory bank has enough contiguous physical memory to store the bitmap? And, of course, as memory sizes increase so does the bootmem bitmap. For a system with 32GB of RAM, the bitmap will require 1MB of that memory. Memblock, on the other hand, can be used immediately as it is based on static arrays large enough to accommodate, at least, the very first memory registrations and allocations. If a request to add or reserve more memory would overflow a memblock array, the array is doubled in size. There is an underlying assumption that, by the time that may happen, enough memory will be added to memblock to sustain the allocation of the new arrays. 

The design of memblock relies on the assumption that there will be relatively few allocation and deallocation requests before the primary page allocator is up and running. It does not need to be especially smart, since its lifetime is limited before it hands off all the memory to the buddy page allocator. 

To ease the pain of transition from bootmem to memblock, a compatibility layer called [nobootmem](<https://elixir.bootlin.com/linux/latest/source/mm/nobootmem.c>) was introduced. Nobootmem provides (most of) the same interfaces as bootmem, but instead of using the bitmap to mark busy pages it relies on memblock reservations. As of v4.17, only five out of 24 architectures are still using bootmem as the only early memory allocator; 14 use memblock with nobootmem. The remaining five use memblock and bootmem at the same time. 

Currently there is ongoing work on enabling the use of memblock with nobootmem on all architectures. Several architectures that use device trees have been converted as a consequence of recent changes in early memory management in the device-tree drivers. There are patches for alpha, c6x, m68k, and nios2 that are already published. Some of them are already merged by the arch maintainers while some are still under review. 

Hopefully, by the 4.20 merge window all architectures will cease using bootmem; after that it will be possible to start a major cleanup of the early memory management code. That work would include removing the bootmem allocator and several kernel configurations associated with it. That, in turn, should make it possible to start moving more early-boot functionality from the architecture-specific subtrees into common code. There is never a lack of problems to solve in the memory-management subsystem.
