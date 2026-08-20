---
title: "Memory: the flat, the discontiguous, and the sparse"
url: https://lwn.net/Articles/789304/
date: "May 27, 2019"
category: "Memory management-Physical memory model"
author: "May 27, 2019 This article was contributed by  Mike Rapoport"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

May 27, 2019

This article was contributed by Mike Rapoport

The physical memory in a computer system is a precious resource, so a lot of effort has been put into managing it effectively. This task is made more difficult by the complexity of the memory architecture on contemporary systems. There are several layers of abstraction that deal with the details of how physical memory is laid out; one of those is simply called the "memory model". There are three models supported in the kernel, but one of them is on its way out. As a way of understanding this change, this article will take a closer look at the evolution of the kernel's memory models, their current state, and their possible future. 

#### FLATMEM

Back in the beginning of Linux, memory was flat: it was a simple linear sequence with physical addresses starting at zero and ending at several megabytes. Each physical page frame had an entry in the kernel's `mem_map` array which, at that time, contained a single `unsigned short` entry for each page that counted the number of references that page had. Soon enough, though, the `mem_map` entries grew to also include `age` and `dirty` counters for the management of swapping. In Linux 1.3.50 the elements of `mem_map` were finally named `struct page`. 

The flat memory map provided easy and fast conversion between a physical page-frame number (PFN) and its corresponding `struct page`; it was a simple matter of calculating an array index. There were changes in the layout of `struct page` to accommodate new usages (the page cache, for example) and to optimize cache performance for the `struct page` accesses. The memory map remained a flat array that was neat and efficient, but it had a major drawback: it couldn't deal well with large holes in the physical address space. Either the part of the memory map corresponding to a hole would be wasted or, as was done on ARM, the memory map would also have holes. 

#### DISCONTIGMEM

Support for the efficient handling of widely discontiguous physical memory was introduced into the memory-management subsystem in 1999 as a part of the effort to make Linux work well on NUMA machines. This code was dependent on the `CONFIG_DISCONTIGMEM` configuration option, so the first memory model that had a name was `DISCONTIGMEM`. 

The `DISCONTIGMEM` model introduced the notion of a memory node, which remains the basis of NUMA memory management. Each node carries an independent (well, mostly) memory-management subsystem with its own free-page lists, in-use page lists, least-recently-used (LRU) information, and usage statistics. Among all these goodies, the node data represented by `struct pglist_data` (or `pg_data_t` for short) contains a node-specific memory map. Assuming that each node has contiguous physical memory, having an array of `page` structures per node solves the problem of large holes in the flat memory map. 

But this doesn't come for free. With `DISCONTIGMEM`, it's necessary to determine which node holds a given page in memory to turn its PFN into the associated `struct page`, for example. Similarly, one must determine which node's memory map contains a `struct page` to calculate its PFN. After a long evolution, starting with the mips64 architecture defining the `KVADDR_TO_NID()`, `LOCAL_MAP_BASE()`, `ADDR_TO_MAPBASE()`, and `LOCAL_BASE_ADDR()` macros for the first time, the conversion of a PFN to the `struct page` and vice versa came to rely on the relatively simple `pfn_to_page()` and `page_to_pfn()` macros defined in [`include/asm-generic/memory_model.h`](<https://elixir.bootlin.com/linux/v5.0/source/include/asm-generic/memory_model.h>). 

`DISCONTIGMEM`, however, had a weak point: memory hotplug and hot remove. The actual NUMA node granularity was too coarse for proper hotplug support, and splitting the node would have created a lot of unnecessary fragmentation and overhead. Remember that each node has an independent memory management structure with all the associated costs; splitting nodes further would increase those costs considerably. 

#### SPARSEMEM

This limitation was resolved with the introduction of `SPARSEMEM`. This model abstracted the memory map as a collection of sections of arbitrary size defined by the architectures. Each section, represented by `[struct mem_section](<https://elixir.bootlin.com/linux/v5.2-rc1/source/include/linux/mmzone.h#L1165>)`, is (as described in the code): ""logically, a pointer to an array of struct pages. However, it is stored with some other magic"". The array of these sections is a meta-memory map which can be efficiently chopped at `SECTION_SIZE` granularity. For efficient conversion between a PFN and `struct page`, several high bits of the PFN are used to index into the sections array. For the other direction, the section number was encoded in the page flags. 

A few months after its introduction into the Linux kernel, `SPARSEMEM` was extended with `SPARSEMEM_EXTREME`, which is suitable for systems with an especially sparse physical address space. `SPARSEMEM_EXTREME` added a second dimension to the sections array and made this array, well, sparse. With `SPARSEMEM_EXTREME`, the first level became pointers to `mem_section` structures, and the actual mem_section objects were dynamically allocated based on the actually populated physical memory. 

Another enhancement to `SPARSEMEM` was added in 2007; it was called [generic virtual memmap support for `SPARSEMEM`](<https://git.kernel.org/linus/8f6aac419bd>), or `SPARSEMEM_VMEMMAP`. The idea behind `SPARSEMEM_VMEMMAP` is that the entire memory map is mapped into a virtually contiguous area, but only the active sections are backed with physical pages. This model wouldn't work well with 32-bit systems, where the physical memory size might approach or even exceed the virtual address space. However, for 64-bit systems `SPARSEMEM_VMEMMAP` is a clear win. At the cost of additional page table entries, `page_to_pfn()`, and `pfn_to_page()` became as simple as with the flat model. 

The last extension of the `SPARSEMEM` memory model is more recent (2016); it was driven by the increased use of persistent-memory devices. To support storing the memory map directly on those devices rather than in main memory, the virtual memory map can use `struct vmem_altmap`, which will provide `page` structures in persistent memory. 

Back in 2005, `SPARSEMEM` was described as a ""newer, and more experimental alternative to 'discontiguous memory'"". The commit that added `SPARSEMEM_VMEMMAP` noted that it ""has the potential to allow us to make `SPARSEMEM` the default (and even the only) option for most systems"". And indeed, several architectures have switched over from `DISCONTIGMEM` to `SPARSEMEM`. In 2008, `SPARSEMEM_VMEMMAP` became the only supported memory model for x86-64, as it was only slightly more expensive than `FLATMEM` but more efficient than `DISCONTIGMEM`. 

Recent memory-management developments, such as memory hotplug, persistent-memory support, and various performance optimizations, all target the `SPARSEMEM` model. But the older models still exist, which comes with the cost of numerous `#ifdef` blocks in the architecture and memory-management code, and a peculiar maze of related configuration options. There is an ongoing work to completely switch the remaining users of `DISCONTIGMEM` to `SPARSEMEM`, but making the change for such architectures as ia64 and mips64 won't be an easy task. And the ARC architecture's use of `DISCONTIGMEM` to represent a "high memory" area that resides below the "normal" memory will definitely be challenging to change.
