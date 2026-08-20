---
title: On pages and folios
url: https://lwn.net/Articles/1064861/
date: "April 24, 2026"
category: "Memory management-Folios; Memory management-struct page"
author: "By Jonathan Corbet April 24, 2026"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 24, 2026

The kernel coverage here at LWN often touches on memory-management topics and, as a result, tends to talk a lot about both pages and folios. As the folio transition in the kernel has moved forward, it has often become difficult to decide which term to use in writing that is meant to be both approachable and technically correct. As this work continues, it will be increasingly common to use "folio" rather than page. This article is intended to be a convenient reference for readers wanting to differentiate the two terms or understand the state of this transition. 

#### Pages

Memory in all but the smallest of computing systems is divided into regularly-sized units called "pages"; the most common page size is 4KB, but there are systems that run with larger page sizes. A page is the smallest unit that the system's hardware units, including the memory-management unit and translation lookaside buffer (TLB) work with. When memory is swapped in or out, or when it is moved between NUMA nodes in a larger system, it is moved in chunks that are an integral number of pages. Pages are thus fundamental to the management of memory in Linux systems. 

This structure is reflected in the way virtual addresses work. On a 64-bit x86 system with a 4KB page size, the upper 52 bits (the "page-frame number" or "PFN") identify the page referred to by the address, while the bottom 12 bits give the offset within the page: 

> ![\[Simple address
structure\]](https://static.lwn.net/images/2026/address-structure1.svg)

Naturally, the full story is a bit more complex than that, starting with the fact that the PFN is really a physical concept, not a virtual one, so the usage here is a bit sloppy — the _real_ page-frame number is only found as part of the address-translation process. Also, the PFN does not usually occupy the full upper 52 bits. Logically, the PFN can be looked at as an index into a large table that stores information about each page, most importantly whether it is resident in RAM and, if so, what its physical address is. In practice, the PFN is split into a maximum of five (as of this writing) nine-bit (on x86-64 systems) indices, each of which is an index into a different table; those tables are then organized into a hierarchy: 

> ![\[More complete address
structure\]](https://static.lwn.net/images/2026/address-structure2.svg)

This structure enables a far more efficient representation of the page tables, and it also, as we will see, comes into play in how huge pages are implemented. On the other hand, it makes for expensive memory access. Every time the CPU encounters a virtual address, it must translate it into a physical address; that means iterating through that series of five levels of page tables, which is going to be slow. To minimize that expense, CPUs maintain a translation lookaside buffer to cache the result of address translations. If a given PFN is in the TLB, the translation will happen quickly; otherwise it will be slow. The TLB is not huge, so a lot of attention goes into ensuring that code uses the TLB efficiently. 

#### The system memory map and `struct page`

The kernel needs to keep track of how every page of memory is being used — that is what "memory management" is all about, after all. To that end, it maintains a large array of [`page` structures](<https://elixir.bootlin.com/linux/v6.19.9/source/include/linux/mm_types.h#L42>), one for each page of physical memory in the system. This structure has been made as small as kernel developers can get it, but it still (typically) requires 64 bytes. As a result, on a system with 4KB pages, the associated `page` structures occupy 1.6% of physical memory. That is a cost that the kernel community has long wanted to reduce. 

The `page` structure has been used throughout the kernel for many years to refer to specific pages of physical memory. At times this ubiquity has proved to be problematic; `struct page` is a core memory-management data structure, but code in other parts of the system often make surprising use of its fields. Current work in the memory-management subsystem is reducing the importance of `struct page`, which may, someday, wither away altogether. 

#### Huge pages

One way to get better use out of the TLB is to have each TLB entry cover a larger area of memory — to make the page size larger, in other words. Most contemporary CPUs implement a huge-page mechanism that does exactly that. In essence, the CPU implements the smallest huge-page size by taking the PTE portion of the page-frame number and turning it into the offset within a larger page instead: 

> ![\[Huge-page address
structure\]](https://static.lwn.net/images/2026/address-structure3.svg)

The entry at the PMD level of the page tables is specially marked to indicate that the PFN stops there and points to a 2MB huge page (again, on x86; other architectures can vary somewhat but the idea remains the same). For this reason, this type of huge page is often referred to as a "PMD-level" (or just "PMD") huge page. By extending the range of a TLB entry from 4KB to 2MB, huge pages can significantly increase the amount of memory that can be addressed without having to go through the whole translation routine. 

Traditionally, applications had to explicitly request huge pages to be able to make use of them. The [transparent huge page (THP)](<https://lwn.net/Articles/423584/>) feature makes it possible for the kernel to provide PMD-level huge pages to user space automatically in situations where it appears that they will help performance. THPs are not always a performance win; they can cause a lot of memory waste if the pages are only sparsely used and they stress the memory-management system more, so they can slow some workloads down. For this reason, the feature ends up being disabled on some systems. 

Larger huge pages exist as well; a PUD-level huge page removes the PMD layer of the page-table hierarchy, yielding a 1GB page size. Such pages can be somewhat unwieldy to work with and can be difficult for the memory-management subsystem to reliably supply, but one common use case is to allocate them for use by virtual machines, which manage them internally, in smaller chunks, as the virtual machine's "physical" memory. 

More recent processors have gained a separate, not-so-huge-page concept. Some x86 processors can mark a TLB entry as covering eight pages, and some Arm processors can perform a similar trick with 16-page chunks. That, again, allows the TLB to cover more of working memory, but without requiring the use of 2MB (or larger) huge pages. The result of these changes is that the sizing of huge pages is becoming more flexible; the term "mTHP" (multi-size transparent huge page) is often used for these smaller page clusters. 

#### Folios

Even in the absence of huge pages, the kernel has long needed to work with larger chunks of physically contiguous memory. The concept of [compound pages](<https://lwn.net/Articles/619514/>) was added to the 2.6.6 kernel release in 2004 as one way of organizing such a chunk; a compound page is a power-of-two-sized group of pages managed, for a period of time, as a single unit. Since a compound page consists of at least two physically contiguous pages, it is represented by an equal number of adjacent `page` structures. The kernel takes advantage of this fact by treating the `page` structure for the first ("head") page as representing the whole set, and storing related information in the `page` structures for the following ("tail") pages. 

Back in 2021, Matthew Wilcox [noticed](<https://lwn.net/Articles/849538/>) that there was a lot of kernel code that could be handed either a compound page or a single ("base") page, with the base page being perhaps located _within_ a compound page. A surprising amount of overhead went into ensuring, in many places in the kernel, that any passed-in `struct page` pointer referred to the head page of a compound page, or to a solitary base page. He decided to improve the kernel's internal APIs to reduce that overhead. The result was the "folio", which was defined as a `struct page` that is known not to represent a tail of a compound page. After some [significant discussion](<https://lwn.net/Articles/868598/>), the initial folio patches were merged for the 5.16 release at the beginning of 2022. 

It became evident fairly quickly, though, that the folio concept has uses far beyond reducing the overhead of supporting compound pages. For decades, kernel developers have contemplated managing memory in larger chunks; the 4KB page size is unchanged from the 1990s, even though the amount of installed memory has grown by several orders of magnitude since then. Contemporary systems have to manage vast numbers of pages, and the associated overhead, in terms of both CPU and memory use, hurts. But attempts to move to larger pages have generally been thwarted by other costs, primarily the lost memory due to internal fragmentation. 

What was needed was a way to deal with memory in variably sized chunks, rather than working with one fixed size (and, perhaps, the vastly larger huge-page sizes). Folios have, since their introduction, been evolving into that way. Over time, areas of the kernel that dealt with pages have been modified to work with variably sized folios instead. 

For example, consider the page cache, which caches portions of files in memory to speed access. The page cache once, true to its name, cached data one page at a time. Now, though, it would be more properly called the "folio cache", with the ability to cache file contents in appropriately sized folios. A small file might well fit within a single-page folio in the page cache, while a much larger file could be cached in a relatively small number of large folios. Making this work required a lot of changes to the memory-management subsystem, the readahead code, and the individual filesystems as well. 

To see how far this transformation has progressed, compare the definitions of `struct address_space_operations`, which (to simplify) describes the functions that move data between the page cache and the underlying persistent storage, from [the 5.16 kernel](<https://elixir.bootlin.com/linux/v5.16.20/source/include/linux/fs.h#L369>) (when folios were introduced but not yet widely used) and [7.0-rc5](<https://elixir.bootlin.com/linux/v7.0-rc5/source/include/linux/fs.h#L403>). The `readpage()` method is now `read_folio()`, many other methods have been changed similarly, and none of them take `struct page` arguments in the current version. These changes were not easy, but they allow the management of the page cache at varying levels of granularity, enable the support of filesystems with block sizes larger than the system page size, and ease the creation of larger (more efficient) I/O operations. 

Anonymous memory for user-space processes has also traditionally been allocated and managed one page at a time. The addition of transparent huge pages helped in some situations, but THPs are too large to be a net performance improvement for many workloads. Instead, mTHPs are easier to work with, waste less memory through internal fragmentation, and can boost performance significantly; folios can represent them nicely within the kernel. The work to make full use of mTHPs is [still ongoing](<https://lwn.net/Articles/1009039/>), and may take a while yet to settle, but mTHPs may prove to be a more generally applicable performance enhancement than PMD-level huge pages for many workloads. 

One significant advantage of moving to folios for both the page cache and anonymous memory is the effect on the kernel's least-recently-used (LRU) lists, which are used to identify which pages (now folios) have not been accessed for a while and should be considered for reclamation. Large numbers of pages lead to extremely long LRU lists, which are more expensive for the kernel to manipulate. Managing folios in those lists makes them shorter, again improving performance. 

Within the kernel, folios are represented by [`struct folio`](<https://elixir.bootlin.com/linux/v7.0-rc5/source/include/linux/mm_types.h#L357>). Since the introduction of folios, this structure has been carefully designed to overlay `struct page` (more correctly, it overlays the first four `page` structures representing a large folio). That work has been done to ensure that the `folio` structures are made up of valid `page` structures, allowing the transition to folios to be implemented incrementally. There will come a time, though, when `struct folio` will become entirely separate from `struct page`, but that will require some fundamental changes to the system's memory map. 

#### Shrinking the memory map

As mentioned above, the kernel's memory map itself takes up a significant amount of memory, which developers would like to see put to better uses. The static nature of the map means that there must be a `page` structure for each physical page, and that said structure must be large enough to handle all of the possible uses to which a page might be put. Making the map more dynamic offers the hope of reducing its memory footprint considerably. 

The eventual plan is to replace `struct page` with an eight-byte memory descriptor; it can be thought of as a pointer to a type-specific structure describing the memory in question, though the real story is a bit more complex. For memory that is organized into folios, the `folio` structure will be the descriptor. Unlike `page` structures, though, there would only need to be a single `folio` structure regardless of how many pages the folio holds. There would still need to be a descriptor entry for each PFN, but the entries in the memory map for the base pages that make up a single folio would all point to a single `folio` structure. There will be other descriptor types for other memory uses, including [slab pages](<https://lwn.net/Articles/871982/>), [page tables](<https://lwn.net/Articles/937839/>), and so on. See [this page](<https://kernelnewbies.org/MatthewWilcox/Memdescs>) for a description of the descriptor types and how they are expected to work. 

The memory-descriptor work is underway, and may take years yet to complete. This sort of transition in a production kernel can be compared to replacing the foundation of building that is in heavy use; it is not a small task. But the fundamental rethinking of the memory-management subsystem that was kicked off by the introduction of folios is moving quickly and has already shown some significant results.
