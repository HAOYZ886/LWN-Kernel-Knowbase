---
title: Reducing page structures for huge pages
url: https://lwn.net/Articles/839737/
date: "December 11, 2020"
category: "Memory management-Compound pages; Memory management-struct page"
author: "By Jonathan Corbet December 11, 2020"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 11, 2020

Kernel development is a constant exercise in reducing overhead; any resources taken by the kernel are not available for the workload that users actually want to run. As part of this, the [`page` structure](<https://elixir.bootlin.com/linux/v5.9.13/source/include/linux/mm_types.h#L29>) used to manage memory has been kept as small as possible. Even so, `page` structures typically take up just over 1.5% of the available memory, which is too much for some users. LWN recently [looked at DMEMFS](<https://lwn.net/Articles/839216/>) as one approach to reduce this overhead, but that is not the only work happening in this area. Two developers are currently working independently on patches to reduce the overhead associated with huge pages in particular. 

#### Memory maps and tail pages

The aforementioned `page` structure holds most of what the kernel knows about every page of memory in the system: how it is used, its position on various lists, the location of its backing store, etc. There is, thus, one `page` structure for each physical page in the system; in common configurations, that means one 64-byte structure for every 4096-byte page. 

In the early days of Linux, the kernel would allocate a simple array of `page` structures that was just large enough to represent the memory installed in the system; that worked because physical memory itself looked like a simple array of pages. Since then, though, life has become rather more complex for the memory-management subsystem. Nodes in NUMA systems have distinct ranges of memory with, possibly, large gaps between them. Memory can be plugged into a system (or removed from it) at run time. Virtualized guests can have memory injected into them (or removed) while they run as well. As a result, the simple, linear model of memory no longer works. 

The kernel has used a few different models for the memory map over time; see [this article](<https://lwn.net/Articles/789304/>) for the full history. The preferred model in current times (for 64-bit systems) is called "sparsemem-vmemmap"; it uses the system's memory-management unit (MMU) to restore the illusion of a simple, linear map (called the "vmemmap"). Specifically, each architecture reserves a portion of the kernel address space for this map; the x86-64 architecture, for example, puts it at `0xffffea0000000000` when four-level page tables are in use. Whenever memory is added to the system (including memory "added" when the kernel discovers it at boot time), a suitable number of `page` structures is allocated and the set is mapped into the vmemmap. Discontiguous chunks of memory can thus be made to appear to be contiguous, simplifying a number of low-level management functions. 

The end result looks vaguely like this: 

> ![\[vmemmap\]](https://static.lwn.net/images/2020/vmemmap1.svg)

On a system with 4096-byte pages and a 64-byte `struct page`, one page of memory will need to be allocated and mapped into the vmemmap array for every 64 `page` structures. Once that is done, the `page` structure for any given page can be found by using its page-frame number as an offset from `vmemmap_base` (on x86 systems). 

[Compound pages](<https://lwn.net/Articles/619514/>) complicate the situation. A compound page is formed when a group of adjacent pages is grouped together into a larger unit. The most common use is for huge pages — larger page sizes implemented by the system's CPU and MMU. The x86-64 architecture, for example, implements 2MB and 1GB huge pages; there can be significant performance benefits from using them. Whenever a huge page is created from a set of single ("base") pages, the associated `page` structures are changed to reflect the compound page that they now represent. 

The first base page in a compound page is called the "head" page, while all of the others are called "tail" pages. A 2MB huge page is thus made up of one head page and 511 tail pages. The `page` structure for the head page is marked as a compound page, and represents the whole set. The `page` structures for the tail pages, instead, contain only a pointer to the head page. (That is not quite true; the first couple tail pages have some metadata about the compound page, but that can be ignored here). 

Thus, of the 512 `page` structures associated with a 2MB huge page, 511 are essentially identical copies of a sign saying "look over there instead". Those structures take up eight pages of memory in their own right, seven of which represent only tail pages and contain identical data. 

#### Trimming wasted `page` structures

Both patch sets described here take the same approach to saving memory. The first out the gate was [this set from Muchun Song](<https://lwn.net/ml/linux-kernel/20201210035526.38938-1-songmuchun@bytedance.com/>), currently in its eighth revision. Song realized that there was no compelling reason to keep all of those pages full of identical `page` structures around, especially given the virtual mapping already used by the vmemmap mechanism. Consider a more compact version of the diagram above: 

> ![\[tail pages\]](https://static.lwn.net/images/2020/vmemmap2.svg)

Here, one 2MB huge page is represented by those eight pages of `page` structures, almost all of which correspond to tail pages and just point to the structure for the head page. 

In a kernel with Song's patch set applied, that structure is changed. Since seven of those eight pages all contain identical pages, they can be replaced with a single page instead; that one page can be mapped seven times to fill out the vmemmap array as before: 

> ![\[shared tail pages\]](https://static.lwn.net/images/2020/vmemmap3.svg)

As far as the rest of the kernel is concerned, nothing has changed; the `page` structure for any given tail page looks the way it did before. But six pages of duplicated data can now be given back to the system for other uses for as long as the compound page continues to exist. In other words, 75% of the memory overhead for this compound page has just been eliminated. 

The savings for 1GB huge pages are even more dramatic; 4094 of the 4096 pages representing tail pages can be eliminated. General-purpose systems tend not to use many 1GB huge pages, but there are situations, including some virtualization workloads, where they are useful. It's not surprising that the prospect of saving nearly 16MB of overhead for each 1GB huge page is attractive to hosting providers. 

Huge pages do not necessarily remain huge forever; they can be returned to the kernel or split up for a number of reasons. When that happens, the full set of `page` structures must be restored. In Song's patch set, this work is deferred to a workqueue so that the necessary pages can be allocated in a relatively relaxed setting. This work adds some compute-time overhead to both the allocation and freeing of huge pages; Song included a set of benchmark results and concluded that this overhead is ""not significant"". There is also overhead added because the patch set disables the use of huge pages for the vmemmap itself; that will evidently be rectified if and when the core patch set is accepted. 

The [alternative patch set](<https://lwn.net/ml/linux-mm/20201208172901.17384-1-joao.m.martins@oracle.com/>) is from Joao Martins. It implements the same idea, eliminating most of the pages containing `page` structures for tail pages. While Song's patch set is focused on main memory, though, Martins's work is specifically for nonvolatile RAM. This memory is always brought online as a separate operation, so there is no need to free existing pages out of the vmemmap; instead, new devices are brought online formatted as huge pages from the beginning. That simplifies the code by eliminating all of the logic to change the vmemmap on the fly, but at the cost of reducing the applicability of the technique. 

One incidental advantage of treating nonvolatile RAM this way, though, is that it eliminates the need to initialize large numbers of `page` structures when a new memory array is attached. That speeds the process of making that memory available to the system considerably. Use of huge pages also significantly accelerates the task of mapping this memory into kernel space, an operation that happens frequently when the DAX direct-access subsystem is in use. 

Both patch sets appear to have advantages and disadvantages. The problem at this point is that the memory-management developers are highly unlikely to be interested in merging both of them. So either one will have to be chosen over the other, or the two patch sets will need to be somehow combined into one that meets everybody's needs. That sort of reconciliation can take time; the process of merging low-level memory-management trickery is not the fastest either. So this work is probably not arriving anytime in the near future, but in the longer term it may well lead to a lot of memory saved on Linux systems.
