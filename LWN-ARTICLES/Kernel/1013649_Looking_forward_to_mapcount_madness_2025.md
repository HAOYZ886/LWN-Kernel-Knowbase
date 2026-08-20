---
title: Looking forward to mapcount madness 2025
url: https://lwn.net/Articles/1013649/
date: "March 17, 2025"
category: "Memory management-Folios"
author: "By Jonathan Corbet March 17, 2025"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 17, 2025

One of the many important tasks that the kernel's memory-management subsystem must handle is keeping track of how pages of memory are mapped into the address spaces of the processes running on the system. As long as mappings to a given page exist, that page must be kept in place. As it turns out, tracking these mappings is harder than it seems it should be, and the move to folios within the memory-management subsystem is adding some complexities of its own. As a follow-up to [the "mapcount madness" session](<https://lwn.net/Articles/974223/>) that he ran at the [2024 Linux Storage, Filesystem, Memory-Management, and BPF summit](<https://lwn.net/Articles/lsfmmbpf2024/>), David Hildenbrand has posted [a patch series](<https://lwn.net/ml/all/20250303163014.1128035-1-david@redhat.com>) intended to improve the handling of mapping counts for folios — but exact accounting remains elusive in some situations. 

In theory, tracking mappings should be relatively straightforward: when a mapping to a page is added, increment that page's mapping count to match. The removal of a mapping should lead to an associated decrement of the mapping count. But huge pages and large folios complicate this scenario; they have their own mapping counts that are, essentially, the sum of the mapping counts for the pages they contain. It is often important to know whether a folio as a whole has mappings, so the separate count is useful, but it brings some complexities. 

For example, one question that the kernel often asks is: how many processes have mapped a given page or folio? There are a number of operations that can be optimized if it is known that a mapping is _exclusive_ — that the page or folio is mapped by a single process. The handling of copy-on-write pages is also hard to execute correctly if the exclusivity of a given mapping is not known; failures on this front have led to [some unpleasant bugs](<https://lwn.net/Articles/895439/>) in the past. For a single page, the exclusivity question is easily enough answered: if the mapping count is one, the mapping is exclusive, otherwise it is shared. That rule no longer applies if mapping counts are maintained at the folio level, though, since the folio-level count will almost never be just one. 

The current scheme also has performance-related problems that folios could maybe help to improve. Mapping a traditional PMD-size huge page is equivalent to mapping 512 base pages; currently, if the entire huge page is mapped, the mapping count of each of its base pages must be incremented accordingly. Incrementing the mapping count on each of those base pages takes time the kernel developers would rather not spend; it would be a lot faster to only keep track of a single mapping count at the folio level. This optimization can only make the exclusivity question even harder to answer, though, especially in the presence of partially mapped folios (where only some of its pages are mapped into an address space). 

Thus, it is not surprising the kernel developers have spent years trying to figure out how to properly manage the mapping counts as memory-management complexity increases. 

#### Make room!

To better track mapping counts at the folio level, Hildenbrand first needed a bit more space in the `folio` structure for some additional information. [`struct folio`](<https://elixir.bootlin.com/linux/v6.13.6/source/include/linux/mm_types.h#L286>) is a bit of a complicated and confusing beast. As a way of facilitating the transition to folio use throughout the kernel, this structure is overlaid on top of [`struct page`](<https://elixir.bootlin.com/linux/v6.13.6/source/include/linux/mm_types.h#L35>), which describes a single page. But folios often need to track more information than can be fit into the tightly packed `page` structure; this is especially true for large folios that contain many pages. 

But, since a large folio does contain many pages — and physically contiguous pages at that — there are some tricks that can be employed. There is no real need to maintain a full `page` structure for every page within a folio, since they are managed as a unit; indeed, eliminating the management of all of those `page` structures is one of the objectives of the folio transition. But those `page` structures exist, laid out contiguously in the system's memory map. So a large folio does not have just one `page` structure's worth of memory at its disposal; it has the `page` structures for all of the component pages. The `page` structures for the "tail pages" — those after the first one — can thus be carefully put to use holding this additional information. 

If one looks at the definition of `struct folio`, it quickly becomes clear that it is larger than a single `page` structure. After the initial fields that overlay the `page` structure for the head page, one will find this: 

```
union {
    	struct {
    	    unsigned long _flags_1;
    	    unsigned long _head_1;
    	    atomic_t _large_mapcount;
    	    atomic_t _entire_mapcount;
    	    atomic_t _nr_pages_mapped;
    	    atomic_t _pincount;
        #ifdef CONFIG_64BIT
    	    unsigned int _folio_nr_pages;
        #endif
    	/* private: the union with struct page is transitional */
    	};
    	    struct page __page_1;
        };
```

This piece of the `folio` structure precisely overlays the `page` structure of the first tail page, assuming such a page exists. It contains information intended to help maintain the mapping count in current kernels, and other relevant fields. There is also a `__page_2` component (not shown) that mainly holds information used by the hugetlbfs subsystem. As a result, the `folio` structure is actually the length of three `page` structures, though most of it is only valid for large (at least four pages) folios. 

As sprawling as this seems, it still lacks the space Hildenbrand needed to better track mapping counts. To be able to handle order-1 (two-page) folios, he needed that space to fit within the page-1 union shown above. So the first six patches of the series are dedicated to shuffling fields around in the `folio` structure, adding a `__page_3` union in the process. The `__page_1` union gains some complexity, but the core of the work is in these new fields: 

```
mm_id_mapcount_t _mm_id_mapcount[2];
        union {
    	mm_id_t _mm_id[2];
    	unsigned long _mm_ids;
        };
```

They will be used to keep better track of the mapping for the folio to which they belong. Describing how that is done requires a bit more background, though. 

#### One, two, or many

So how does all of this work help to improve the tracking of the mapping counts for large folios that may be shared between multiple processes and which can be partially mapped in any one of them? The starting point is the [`mm_struct` structure](<https://elixir.bootlin.com/linux/v6.13.6/source/include/linux/mm_types.h#L803>) that represents a process's address space. Any time a folio is mapped, that mapping will belong to a specific process, and thus a specific `mm_struct` structure. So the question of whether a folio is exclusively mapped comes down to whether all of its mappings belong to the same `mm_struct`. It is a simple matter of tracking which `mm_struct` structures hold mappings to the folio. 

Of course, there could be thousands of those structures containing such mappings; consider that almost every process in the system will have the C library mapped, for example. Tracking all of those mappings without consuming a lot of time and memory would not be an easy task. But it is not really important to track every mapping to something like the C library; the purpose here is to stay on top of the folios that are exclusively mapped, and thus don't have all those mappings. 

The `_mm_id` array that was added to page 1 of the `folio` structure is intended to serve this purpose; it can track up to two `mm_struct` structures that have mappings to the folio. The most straightforward way to do that would be to just store pointers to those `mm_struct` structures, but space in the `folio` structure is still at a premium. So, instead, a shorter "mm ID" is assigned to each `mm_struct`, using the kernel's [ID allocator](<https://docs.kernel.org/core-api/idr.html>) subsystem. 

When a folio is first created, both `_mm_id` entries are set to `MM_ID_DUMMY`, indicating that they are unused. When the time comes to add a mapping, the kernel will search `_mm_id` for the appropriate mm ID, then increment the associated `_mm_id_mapcount` entry to record the new mapping(s). So, for example, if eight pages within a folio are mapped into the address space, the count will be incremented by eight to match. If the mm ID does not have an entry in `_mm_id`, the kernel will look for an `MM_ID_DUMMY` entry to use for this `mm_struct`, then start tracking the mappings there. 

The kernel is now maintaining multiple mapping counts for this folio. The `_large_mapcount` field of the `folio` structure continues to count all of the mappings to the folio from any address space, as it does in current kernels. But there is also the `_mm_id_mapcount` count for each `mm_struct` tracking the number of mappings associated with that specific structure. The question of whether the folio is mapped exclusively is now easy to answer: if one of the `_mm_id_mapcount` counters is equal to `_large_mapcount`, then all of the mappings belong to the associated `mm_struct` and the kernel knows that the mapping is exclusive. Otherwise, the mapping is shared. 

The ability to track two `mm_struct` structures handles the most common case of short-term shared mappings — when a process calls [`clone()`](<https://man7.org/linux/man-pages/man2/clone.2.html>) to create a new child process. That new process will use the second `_mm_id` slot for the mapping that is now shared between the parent and the child. If, as often happens, the child calls [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>) to run a new program, the shared mapping will be torn down, the child's `_mm_id` slot will be released, and the kernel will know that the folio is, again, mapped exclusively. 

There is just one tiny gap in this mechanism, though: what happens when a third process comes along and maps the folio? There will be no `_mm_id` slot available for it, so its mapping(s) cannot be tracked. Should this happen, the kernel will set a special bit in the `folio` structure indicating that it no longer has a handle on where all the mappings to the folio come from, and will treat it as being shared. This could result in the kernel mistakenly concluding that a folio is mapped shared when it is mapped exclusively; the consequence will be worse performance, but no lack of correctness. If enough processes unmap the folio, there could come a time when `_large_mapcount` again aligns with one of the `_mm_id_mapcount` counts, and the kernel will once again know that the folio is mapped exclusively. 

#### Per-page mapcounts and more

The result of all this work is that the kernel has a better handle on whether any given folio is mapped exclusively or shared, though it may still occasionally conclude that a folio is shared when it is not. But that was not the only objective of this work; Hildenbrand also would like to do away with the overhead of maintaining the per-page mapping counts in large folios. The final part of the patch series is an implementation of that goal; at the end, the per-page counts are no longer used or maintained. 

The most significant consequence of dropping the per-page mapping counts appears to be making some of the memory-management statistics provided by the kernel (the various resident-set sizes, for example) a bit fuzzier. Hildenbrand suggests that this imprecision should not be a problem, but he also acknowledges that it will take time to see what the implications really are. To avoid surprises during that time, there is a new configuration parameter, `CONFIG_NO_PAGE_MAPCOUNT`, that controls whether these changes are effective. This work is considered experimental enough that, at this point, Hildenbrand does not want to have it enabled by default in production kernels. 

There will be a desire to do that at some point, though; dropping the per-page map counts can make a `clone()` call up to 20% faster for some workloads, according to performance results included in the patch cover letter. 

Meanwhile, this work enables another optimization with regard to how some transparent huge pages are used after a process forks. In current kernels, if the huge page (folio) is mapped at the base-page level ("PTE mapped"), it will not be reused after the fork. As the use of transparent huge pages — and, especially, in multi-size huge pages that _must_ be PTE mapped — grows, reusing those huge pages will become increasingly important. Now, with the per-`mm_struct` mapping counts, the kernel can tell when a process has exclusive access to the huge page and can continue to use it as such. This reuse yields significant improvements in some benchmark results. 

The use of large folios is expected to grow in the future; they are a more efficient way to manage much of the memory that any given process uses. So it is important to optimize that case as much as possible. Hildenbrand's patch set makes some steps in that direction while addressing a thorny problem that has resisted solution for years. These changes are currently in the linux-next repository, so there is a reasonable possibility that they could land in the mainline during the 6.15 merge window. If so, the [2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), which will be concurrent with that merge window, may be the last to feature a "mapcount madness" session.
