---
title: Freezing out the page reference count
url: https://lwn.net/Articles/1000654/
date: "December 6, 2024"
category: "Memory management-struct page; Releases-6.14"
author: "By Jonathan Corbet December 6, 2024"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 6, 2024

The [`page` structure](<https://elixir.bootlin.com/linux/v6.12.1/source/include/linux/mm_types.h#L35>) sits at the core of the kernel's memory-management subsystem (for now), and a key part of that structure is its reference count, stored in `refcount`. The page reference count tells the kernel how many users a given page has and when it can be freed. That count is not needed for every page in the system, though. Matthew Wilcox has recently [resurrected](<https://lwn.net/ml/all/20241125210149.2976098-1-willy@infradead.org>) an [old patch set](<https://lwn.net/ml/linux-mm/20220809171854.3725722-1-willy@infradead.org/>) that expands the concept of a "frozen" page — one that lacks a meaningful reference count — to the immediate benefit of the slab allocator but in the service of a longer-term goal as well. 

The kernel is in the business of managing resources that are shared between multiple users. For example, anonymous (data) pages and file-backed pages can both be mapped into the address space of one or more processes; each mapping increases the relevant page's reference count, ensuring that the page stays around as long as it is needed. Reference counts can be a relatively efficient way to manage object lifecycles, but they are not free. Frequent reference-count changes can cause cache-line bouncing, and the atomic operations needed to change a reference count are relatively expensive. So there is value in _not_ using a reference count when the opportunity arises. 

In the case of the `struct page` reference count, its use is so deeply ingrained within the memory-management subsystem that it is maintained for almost all pages in the system — even those for which it is not needed. One case, in particular, is the slab allocator, which allocates groups of pages, splits them into smaller objects, and hands those objects out on request. A call to `kmalloc()` is the most common way to get memory from the slab allocator. Since it must track the status of each of the sub-objects contained within a page, the slab allocator knows whether the page as a whole is in use or not; it does not need a separate reference count for that purpose. 

Even so, pages given to the slab allocator are reference-counted. The overhead of maintaining that reference count may not seem like much, but it can add up, especially under workloads that exercise the slab allocator heavily. Given the amount of effort that goes into optimizing the kernel for even tiny gains, eliminating this potentially costly atomic operation seems like a worthwhile goal on its own. 

#### Frozen pages

Wilcox's patch set expands on the notion of a "frozen" page — one for which the reference count has been frozen (with a value of zero) and is not in use. In current kernels, frozen pages are only used in the [hugetlbfs subsystem](<https://docs.kernel.org/admin-guide/mm/hugetlbpage.html>), which maintains a reserve of huge pages for application use. In that case, frozen pages are used to avoid manipulating reference counts while assembling larger pages, but the concept is more widely useful. 

In current kernels, a page's reference count is initialized with a call to [`set_page_refcounted()`](<https://elixir.bootlin.com/linux/v6.12.1/source/mm/internal.h#L440>), which sets the count to one. This initialization is done deeply within the page-allocation paths; there are only three call sites in the entire kernel. The bulk of the patch set is, perhaps surprisingly, dedicated to adding many more of those call sites. In short, the `set_page_refcounted()` calls are pushed down the call stack into the callers of the low-level allocation functions. This change enables the existence of allocation paths that never set the reference count at all. Still, it may seem like a counterproductive change; `set_page_refcounted()` turns into a single assignment instruction (no atomic operations are needed for the initial reference), and the code was arguably cleaner before this change. There are reasons to do things this way, though, as we will see. 

Once those calls have all been pushed down, a new allocation function (a macro, in truth) is added: 

```
struct page *alloc_frozen_pages(gfp_t gpf_flags, unsigned int order);
```

Along with that, the existing [`free_unref_page()`](<https://elixir.bootlin.com/linux/v6.13-rc1/source/mm/page_alloc.c#L2641>) is renamed to `free_frozen_pages()`. The latter function is where the bigger savings is to be found. The normal function for freeing pages ([`__free_pages()`](<https://elixir.bootlin.com/linux/v6.13-rc1/source/mm/page_alloc.c#L4811>) or one of its callers) works by decrementing the reference count of the pages passed to it; it only actually frees the pages when the count goes to zero. `free_frozen_pages()` can avoid the atomic decrement-and-test operation on the reference count, since it knows that there are no other references to the page. 

Once that work is done, the final step is [a small patch](<https://lwn.net/ml/all/20241125210149.2976098-16-willy@infradead.org>) causing the slab allocator to use `alloc_frozen_pages()` and `free_frozen_pages()` rather than `alloc_pages()` and `__free_pages()`. The unneeded atomic operation is eliminated, and the slab allocator is presumably that much faster — though no benchmark results have been included to quantify that. 

#### The future is glorious

Performance patches normally should include benchmarks, but the performance improvement here is really more of a side effect; the real purpose driving this work is something different. Since the beginning of the [folio transition](<https://lwn.net/Articles/849538/>) some years ago, one of the end goals has been the reduction of the size of `struct page`. This structure is as small as developers could make it but, since one of them exists for every page in the system, `page` structures still end up occupying roughly 1.5% of the memory in the system. Reclaiming some of that overhead for productive use is an attractive prospect. 

The long-term goal is to reduce `struct page` to a single, 64-bit [memory descriptor](<https://kernelnewbies.org/MatthewWilcox/Memdescs>) that indicates how each page is being used and contains a pointer to a structure with the information needed for that usage. For example, slab pages can be [described by `struct slab`](<https://lwn.net/Articles/881039/>); that structure exists in current kernels, but is carefully crafted as an overlay on top of `struct page`. In a future world, a single `struct slab` could exist for an entire folio of slab pages, reducing the memory-management overhead for those pages; a pointer to that structure would be placed into each of the relevant memory descriptors. 

In current kernels, `struct slab`, being an overlay on `struct page`, contains the `refcount` field; that will remain true even if Wilcox's patch set is merged. But, in a future where `struct slab` no longer needs to overlay `struct page`, that reference count can be removed, shrinking `struct slab`. And that is, indeed, the future toward which Wilcox is looking: 

> This patchset is also a step towards the Glorious Future in which struct page doesn't have a refcount; the users which need a refcount will have one in their per-allocation memdesc. 

Many steps toward the ""Glorious Future"" have already been taken; some of the type-specific memory descriptors (such as `struct slab`) have already been merged, and others are in the works. There has been an ongoing effort to move information out of `struct page` and into `struct folio` where appropriate. Other projects, like [the removal of the `index` member](<https://lwn.net/ml/all/Z09hOy-UY9KC8WMb@casper.infradead.org>) from `struct page`, are ongoing. This member, which is used for page-cache pages, tells the kernel the offset of the page within the file it represents on disk; it is being shifted over to the `folio` structure. That work depends, in turn, on the [zswap memory-descriptor transition](<https://lwn.net/ml/all/20241205175000.3187069-1-willy@infradead.org>), which is also in progress. There are many other steps yet to be taken, but at the conclusion of this work, `struct page` should mostly have just withered away. 

So, while the removal of a single atomic operation from the slab allocator's page-freeing path is only so exciting, this patch series is rather more interesting as a piece in the larger memory-descriptor project. The core of the kernel's memory-management subsystem is going through a period of radical (and much-needed) change that will make it more efficient, flexible, and maintainable in the long term, and most users are not even noticing. The task is somewhat like exchanging the foundation underneath a skyscraper while it remains open for business; in the kernel community, though, it's just development as usual.
