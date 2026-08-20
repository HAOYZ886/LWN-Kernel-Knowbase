---
title: An introduction to compound pages
url: https://lwn.net/Articles/619514/
date: "November 11, 2014"
category: "Memory management-Compound pages"
author: "By Jonathan Corbet November 11, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 11, 2014

Your editor was digging through a patch set that makes changes involving compound pages when he realized that his understanding of these pages was a bit on the weak side. After some time digging through the source to rectify that situation, a thought surfaced: the world must be full of people wishing they knew more about compound pages. For all of you whose list of desired lifetime accomplishments includes a better understanding of this subject, here is a quick introduction to compound pages in the Linux kernel. 

A compound page is simply a grouping of two or more physically contiguous pages into a unit that can, in many ways, be treated as a single, larger page. They are most commonly used to create huge pages, used within [hugetlbfs](<https://lwn.net/Articles/374424/>) or the [transparent huge pages](<https://lwn.net/Articles/423584/>) subsystem, but they show up in other contexts as well. Compound pages can serve as anonymous memory or be used as buffers within the kernel; they cannot, however, appear in the page cache, which is only prepared to deal with singleton pages. 

Allocating a compound page is a matter of calling a normal memory allocation function like `alloc_pages()` with the `__GFP_COMP` allocation flag set and an order of at least one. It is not possible to create an order-zero (single-page) compound page due to the way compound pages are implemented. (The "order" of an allocation is the base-2 logarithm of the number of pages to allocate; zero thus corresponds to a single page, one to two pages, etc.). 

Note that a compound page differs from the pages returned from a normal higher-order allocation request. A call like: 

```
pages = alloc_pages(GFP_KERNEL, 2);  /* no __GFP_COMP */
```

will return four physically contiguous pages, but they will not be a compound page. The difference is that creating a compound page involves the creation of a fair amount of metadata; much of the time, that metadata is unneeded so the expense of creating it can be avoided. 

So what does that metadata look like? Since most of it is stored in the associated `page` structures, one can assume that [it's complicated](<https://lwn.net/Articles/565097/>). Let's start with the page flags. The first (normal) page in a compound page is called the "head page"; it has the `PG_head` flag set. All other pages are "tail pages"; they are marked with `PG_tail`. At least, that is the case on systems where page flags are not in short supply — 64-bit systems, in other words. On 32-bit systems, there are no page flags to spare, so a different scheme is used; all pages in a compound page have the `PG_compound` flag set, and the tail pages have `PG_reclaim` set as well. The `PG_reclaim` bit is normally used by the page cache code, but, since compound pages cannot be represented in the page cache, that flag can be reused here. 

Code dealing with compound pages need not worry about the different marking conventions, though. No matter which convention is in use, a call to `PageCompound()` will return a true value if the passed-in page is a compound page. Head and tail pages can be distinguished, should the need arise, with `PageHead()` and `PageTail()`. 

Every tail page has a pointer to the head page stored in the `first_page` field of `struct page`. This field occupies the same storage as the `private` field, the spinlock used when the page holds page table entries, or the `slab_cache` pointer used when the page is owned by a slab allocator. The `compound_head()` helper function can be used to find the head page associated with any tail page. 

There is a bit of information describing the compound page as a whole: the order (size) of the page, and a destructor used to return the page to the system when it is no longer needed. One might first think to store that information in the head page's `struct page`, but there is no room for it there. Instead, the order is stored in the `lru.prev` field in the `page` structure for the first _tail_ page. While unions are used for many of the overlaid fields in `struct page`, here the order is simply cast into a pointer type before being stored in a pointer field. Similarly, a pointer to the destructor is stored in the `lru.next` field of the first tail page's `struct page`. This extension of compound-page metadata into the second `page` structure is why compound pages must consist of at least two pages. 

Incidentally, there are only two compound page destructors declared in the kernel. By default, `free_compound_page()` is used; all it does is return the memory to the page allocator. The hugetlbfs subsystem, though, uses `free_huge_page()` to keep its accounting up to date. 

In most cases, compound pages are unnecessary and ordinary allocations can be used; calling code needs to remember how many pages it allocated, but otherwise the metadata that would be stored in a compound page is unneeded. A compound page is indicated, though, whenever it is important to treat the group of pages as a whole even if somebody references a single page within it. Transparent huge pages are a classic example; if user space attempts to change the protections on a portion of a huge page, the entire huge page will need to be located and broken up. Various drivers also use compound pages to ease the management of larger buffers. 

And that is pretty much everything that distinguishes a compound page from an ordinary, higher-order allocation. Most developers will not encounter compound pages in their area of the kernel. In cases where it is truly necessary to treat a set of pages as a single unit, though, compound pages may well be part of the solution toolkit.
