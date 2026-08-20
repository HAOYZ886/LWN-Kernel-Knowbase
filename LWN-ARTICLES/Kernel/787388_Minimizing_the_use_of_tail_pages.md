---
title: Minimizing the use of tail pages
url: https://lwn.net/Articles/787388/
date: "May 4, 2019"
category: "Memory management-Compound pages"
author: "By Jonathan Corbet May 4, 2019 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 4, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

[Compound pages](<https://lwn.net/Articles/619514/>) are created by the kernel as a way of combining a number of small pages into a single, larger unit. Such pages are implemented as a single "head page" at the beginning, followed by a number of "tail pages". Matthew Wilcox has concluded that it would be beneficial to minimize the use of tail pages in the kernel; he ran a session during the memory-management track at the 2019 Linux Storage, Filesystem, and Memory-Management Summit to explore how that could be done. The discussion ranged widely, veering into the representation of DMA I/O operations, but few hard conclusions were reached. 

The most common use for compound pages is to represent huge pages, as created by the transparent huge pages or hugetlbfs features. The slab allocators can also create compound pages to satisfy larger allocations. Since compound pages can be thought of as simply being larger pages, they can usually be managed using just the head page. There are times, though, when the tail pages come into play. Most often, something (a page fault, perhaps) will create a reference within a larger page; the tail page is then used to locate the head page, which is where most of the relevant information is stored. 

To make things as transparent as possible, many places in the kernel use the [`compound_head()`](<https://elixir.bootlin.com/linux/v5.0.11/source/include/linux/page-flags.h#L140>) function to ensure that they are looking at a head page, even when the page [![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2019/lsfmm/MatthewWilcox-sm.jpg)](<https://lwn.net/Articles/787414/>) in question may not be a compound page. That is overhead that, perhaps, does not need to be incurred. There is a `READ_ONCE()` memory barrier in it, Wilcox pointed out; even if the overhead is small, it will be greater than zero and nice to get rid of. 

There are a number of kernel functions that can yield a reference to a tail page, including `virt_to_page()`, `phys_to_page()`, and `find_get_entry()`. If those functions were to, instead, give a pointer to the head page, the calls to `compound_head()` could be eliminated. In the case of `virt_to_page()`, one can simply call `virt_to_head_page()` instead when one knows that the result could otherwise be a tail page — problem solved. There is no `phys_to_head_page()`, but perhaps that should be added. 

A trickier problem is the page cache; if a transparent huge page is stored there, all of the tail pages are stored along with the head page. Wilcox has a patch to store only the head page, making things far more efficient in general, but there are a lot of related functions that can return tail pages; these include `find_get_page()`, `find_get_pages_range()`, and others. There is thus a need for replacements that only return head pages, followed by the usual effort to track down and fix all of the callers. 

Even trickier is the problem of dealing with page tables; this is what Wilcox described as the "controversial bit". He would like to introduce a replacement for `get_user_pages()` called `get_user_head_pages()`; it would only return the head pages for the region of memory mapped into the kernel. Callers would not be able to make assumptions about the size of the pages, and would have to be more careful in how they work with those pages. This function would, as a result, be harder to use than the function it replaces (which is already not entirely easy), it is more complex, and there are a lot of callers to fix. He is not sure when or even if such a transition would ever be completed. Additionally, as Kirill Shutemov pointed out, callers may really want the tail page sometimes, so it may not be possible to make this change at all. 

A second option is thus worth considering. Wilcox asserted that most callers of `get_user_pages()` have, as their real goal, the creation of a scatter/gather list for DMA I/O operations — an assertion that Christoph Hellwig immediately disputed. Presenting a list of files containing `get_user_pages()` calls, Wilcox asked an attendee to pick one at random; digging into that file, he showed that the results of the call were, indeed, used to create a scatter/gather list. Hellwig then went into a characteristic exposition on why the current `struct scatterlist` data structure is the wrong solution to the problem, since it mixes DMA information with the host representation of the memory and creates confusion when memory areas are coalesced on the DMA side. 

A replacement for the current scatter/gather list representation may be on the horizon, but that is somewhat orthogonal to the idea that Wilcox was getting at here. Rather than having kernel code call `get_user_pages()` and using the result to create a scatter/gather list, it might be better to implement a `get_user_sg()` function that would do the full job. That function could then hide any dealings with tail pages while simultaneously simplifying callers. 

If `struct scatterlist` is to be eliminated, `get_user_sg()` could still be implemented in some form. The leading contender for a replacement structure appears to be [`struct biovec`](<https://elixir.bootlin.com/linux/v5.0.11/source/include/linux/bvec.h#L27>). This structure has its origins in the block layer, but is widely used in the rest of the kernel at this point; the networking code was described as the biggest remaining holdout. A `biovec` can describe the host side of an I/O operation; the device side would need a different structure that, perhaps, is more specific to the hardware involved. 

In any case, almost everything discussed in this session is theoretical until patches appear on the mailing lists. Until then, the kernel will continue to deal with tail pages (and scatter/gather lists) the way it has for years.
