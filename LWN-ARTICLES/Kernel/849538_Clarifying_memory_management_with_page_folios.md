---
title: Clarifying memory management with page folios
url: https://lwn.net/Articles/849538/
date: "March 18, 2021"
category: "Memory management-Folios"
author: "By Jonathan Corbet March 18, 2021"
---

By **Jonathan Corbet**  
March 18, 2021

Memory management generally works at the level of pages, which typically contain 4,096 bytes but may be larger. The kernel, though, has extended the concept of pages to include compound pages, which are groups of contiguous single pages. That, in turn, has made the definition of what a "page" is a bit fuzzy. Matthew Wilcox has been working since last year on a concept called "page folios" which is meant to bring the picture back into focus; whether the memory-management community will accept it remains unclear, though. 

At the lowest level, pages are a concept implemented by the hardware; the tracking of memory and whether it is present in RAM or not is done at page granularity. Any given CPU architecture may offer a limited selection of page sizes, but one "base" page size must be chosen, and the most common choice remains 4,096 bytes — the same as it was when the first Linux kernels were released 30 years ago. 

The kernel, though, often has reason to work with memory in larger chunks. One example is the management of "huge pages" which, once again, are implemented by the hardware. The x86 architecture, for example, can work with 2MB huge pages, and there are performance advantages to using them where they are applicable. The kernel will also allocate groups of pages in other sizes, though, typically for DMA buffers or other uses where a set of physically contiguous pages is needed. This sort of grouping of pages is known as a "compound page" in the kernel. 

Every base page of memory managed by the kernel is represented by a [`page` structure](<https://lwn.net/Articles/565097/>) in the system memory map. When a compound page is created out of a set of base pages, the `page` structure for the first page in the set (the "head page") is specially marked to make its compound nature explicit. The other information in that structure refers to the compound page as a whole. All of the other pages (the "tail pages") are marked as such, with a pointer to the `page` structure for the associated head page. See [this article](<https://lwn.net/Articles/619514/>) for details on how compound pages are organized. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

This mechanism makes it easy to go from the `page` structure of a tail page to the head page for the compound page. Many interfaces within the kernel make use of that feature, but it creates a fundamental ambiguity: if a function is passed a pointer to a `page` structure for a tail page, is it expected to act on that tail page or on the compound page as a whole? Or, as Wilcox put it in [the first posting of the folio series](<https://lwn.net/ml/linux-kernel/20201216182335.27227-1-willy@infradead.org/>) in December: 

> A function which has a struct page argument might be expecting a head or base page and will BUG if given a tail page. It might work with any kind of page and operate on PAGE_SIZE bytes. It might work with any kind of page and operate on page_size() bytes if given a head page but PAGE_SIZE bytes if given a base or tail page. It might operate on page_size() bytes if passed a head or tail page. We have examples of all of these today. 

(`PAGE_SIZE` is the size of a base page, while `page_size()` returns the full size of a — possibly compound — page.) There does not seem to be an extensive history of bugs resulting from this particular API, but an interface that is this poorly defined seems likely to encourage problems sooner or later. 

In an attempt to clarify the situation, Wilcox has come up with the concept of a "page folio", which is really just a `page` structure that is guaranteed not to be a tail page. Any function accepting a folio will operate on the full compound page (if, indeed, it is a compound page) with no ambiguity. The result is greater clarity in the kernel's memory-management subsystem; as functions are converted to take folios as arguments, it will become clear that they are not meant to operate on tail pages. 

When Wilcox first posted this patch series, though, he emphasized a different benefit from the change. Any function that might be passed a tail page, but which must operate on the full compound page containing that tail page, must exchange any pointers to tail-page `page` structures for pointers to the head page instead. That is typically done with a call to: 

```
struct page *compound_head(struct page *page);
```

This function is relatively cheap, but it may be called many times over the course of a single operation on a page. That makes the kernel bigger (since it's an inline function) and slows things down. A function that accepts a folio, instead, knows that it is not dealing with a tail page; thus it need not call `compound_head()`. That saves both time and memory. 

The folio type itself is defined as a simple wrapper structure: 

```
struct folio {
            struct page page;
        };
```

From there, a new set of infrastructure is built up. For example, `get_folio()` and `put_folio()` will manage references to the folio much like `get_page()` and `put_page()`, but without the unneeded calls to `compound_head()`. A whole set of higher-level functions follows from there. Much of the real work, though, will be in converting various kernel subsystems to use the new type; Wilcox didn't sugarcoat the nature of that task: 

> This is going to be a ton of work, and massively disruptive. It'll touch every filesystem, and a good few device drivers! But I think it's worth it. 

By the time the [fourth version of this patch set](<https://lwn.net/ml/linux-kernel/20210305041901.2396498-1-willy@infradead.org/>) was posted on March 5, the core patches and the conversions (which Wilcox didn't post) added up to about 100 commits, which is a fair amount to review. 

Perhaps as a result of the size of the patch series, the previous postings did not elicit that much discussion. In response to the latest one, though, Andrew Morton [took a look](<https://lwn.net/ml/linux-kernel/20210313123658.ad2dcf79a113a8619c19c33b@linux-foundation.org/>) and was worried by what he saw: 

> Geeze it's a lot of noise. More things to remember and we'll forever have a mismash of `page' and `folio' and code everywhere converting from one to the other. Ongoing addition of folio accessors/manipulators to overlay the existing page accessors/manipulators, etc. 
> 
> It's unclear to me that it's all really worth it. 

Hugh Dickins, too, [expressed a lack of enthusiasm](<https://lwn.net/ml/linux-kernel/alpine.LSU.2.11.2103131842590.14125@eggly.anvils/>) for this work. On the other hand, [Kirill Shutemov](<https://lwn.net/ml/linux-kernel/20210315115501.7rmzaan2hxsqowgq@box/>) and [Michal Hocko](<https://lwn.net/ml/linux-kernel/YE9VLGl50hLIJHci@dhcp22.suse.cz/>) both expressed support for it, in concept at least. Dave Chinner [said](<https://lwn.net/ml/linux-kernel/20210315222708.GA349301@dread.disaster.area/>) that ""this abstraction is absolutely necessary"" for filesystem developers, especially if and when the page cache gains the ability to manage compound pages of multiple sizes. 

So, in other words, there is currently no consensus among the core developers regarding whether this work improves the kernel or not. That may change over time as more people look at it and its advantages (or the lack thereof) become more clear. But change tends to happen slowly in the memory-management subsystem in general, even when the patch set in question is not so large and messy. One should also bear in mind that there is an inevitable discussion on naming to be had; it is already clear that "folio" is not popular, though alternatives are currently thin on the ground. One conclusion is thus clear: the kernel may well get folios or something like them, but it seems unlikely to happen soon.
