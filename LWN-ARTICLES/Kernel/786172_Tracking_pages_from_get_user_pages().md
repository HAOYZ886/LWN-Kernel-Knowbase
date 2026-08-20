---
title: Tracking pages from get_user_pages()
url: https://lwn.net/Articles/786172/
date: "April 18, 2019"
category: "Memory management-get user pages"
author: "By Jonathan Corbet April 18, 2019"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 18, 2019

As has been recently [discussed](<https://lwn.net/Articles/784574/>) here, developers for the filesystem and memory-management subsystems have been grappling for years with the problems posed by the `get_user_pages()` mechanism. This function maps memory into the kernel's address space for direct access by the kernel or peripheral devices, but that kind of access can create confusion in the filesystem layers, which may not be expecting that memory to be written to at any given time. A new [patch set](<https://lwn.net/ml/linux-kernel/20190411210834.4105-1-jglisse@redhat.com/>) from Jérôme Glisse tries to chip away at a piece of the problem, but a complete solution is not yet in view. 

The problem with `get_user_pages()` is relatively simple to understand: filesystems go to great lengths to track whether any given file page in memory is in a clean state — whether it matches the data in persistent storage — or not. When necessary, they use the memory-management subsystem to prevent changes to specific pages so that those pages can be written in a known state; once a page is clean it can be made writable again. Pointers to pages obtained by `get_user_pages()` bypass this mechanism, though; peripheral devices remain able to write data to those pages at any time. A poorly timed write can lead to data corruption or kernel crashes, neither of which is likely to be the behavior the user of the system is hoping for. Things get even more complicated if the pages in question are stored in persistent memory. 

The above-linked article covered a nascent plan to track pages that have references created by `get_user_pages()`, perhaps by playing tricks with the page reference counts. Recent [reference-counting changes](<https://lwn.net/Articles/786044/>) might just have thrown a spanner into that works, though; the implementation of this plan has not yet been posted. Glisse's patch set is intended to work with it once it is around, though; in particular, it is designed to get the block layer to do its part to ensure that the tracking is correct. To do so, it creates a new mechanism to track the origin of pages that are given to the block layer with I/O requests. 

#### A new bio_vec

Filesystems generate I/O operations in response to file read and write requests; those operations are represented by [`struct bio`](<https://elixir.bootlin.com/linux/v5.0/source/include/linux/blk_types.h#L140>); it is common usage to call one of these structures a "BIO". Within a BIO, the data to be transferred is represented by [`struct bio_vec`](<https://elixir.bootlin.com/linux/v5.0/source/include/linux/bvec.h#L27>): 

```
struct bio_vec {
    	struct page	*bv_page;
    	unsigned int	bv_len;
    	unsigned int	bv_offset;
        };
```

Of note here is the `bv_page` field, which points to a `page` structure for the memory page holding the data of interest. For normal buffered I/O, that page is probably owned by the kernel and resident in the page cache; there is no need to call `get_user_pages()` to get that pointer. For some types of operations, though, including direct I/O requests, that page may belong to a user-space process. Executing such requests requires calling `get_user_pages()` to ensure that the page is locked in memory and to obtain a pointer to its `page` structure. 

The purpose of Glisse's patch set is to enable filesystems and the block layer to track the origin of pages found in these `bio_vec` structures. The approach taken to get there is not entirely obvious; it starts by changing the `bv_page` member to: 

```
unsigned long   bv_pfn;
```

The pointer to the `page` structure has been changed to an integer page-frame number (or PFN). To simplify the picture a bit, one can imagine that the kernel maintains a big array of `page` structures, one for each page of memory in the system. As a `struct page` pointer, `bv_page` pointed directly to one entry in that array. A page-frame number, instead, can be thought of as an integer index into that array. 

For the most part, the two ways of representing a page are equivalent, but there is a difference that is being exploited here. A pointer is a full 64-bit value; it leaves no space to stuff in an extra bit of information or two. (That is not strictly true; if one assumes certain alignment restrictions, the low-order bit(s) might be usable for other purposes. In fact, the kernel often uses the low-order bits of pointers to store related information). Page-frame numbers, instead, can be thought of as pointers with the bottom 12 bits removed since they do not track offsets within a page. They require less space, and thus provide more space to cram in other data. 

The patch set uses the highest-order bit in the PFN to store a flag called `BVEC_PFN_GUP`; that bit will be set if the page in question has been obtained through `get_user_pages()`. Getting there is not a trivial task, though. In current kernels, code that manipulates `bio_vec` structures will access the `bv_page` field directly; all of those accesses had to be changed to use helper functions instead. That required [a large patch](<https://lwn.net/ml/linux-kernel/20190411210834.4105-6-jglisse@redhat.com/>) touching 92 files all over the kernel. Even then, the new information can only be stored if creators of BIOs make a note of where their pages came from. That requires changing functions like `[bio_add_page()](<https://elixir.bootlin.com/linux/v5.0/source/block/bio.c#L809>)` to have an `is_gup` parameter describing the origin of each page — and changing every caller as well. [That patch](<https://lwn.net/ml/linux-kernel/20190411210834.4105-11-jglisse@redhat.com/>) touches 56 files. 

Toward the end of a 15-part patch series, the block layer is able to keep track of which pages given to it originally came from a `get_user_pages()` request. All of this work appears to have been done for one reason: so that the block layer can properly release references to those pages. 

#### Properly putting pages

The tracking mechanism mentioned at the beginning of this article is meant to keep filesystems (and other kernel code) informed about which pages have references created by `get_user_pages()`. One piece of that puzzle is keeping track of when one of those references is released; that is done by requiring a call to the proposed `put_user_page()` function to release a reference rather than `put_page()`. This function has not yet been merged (it is likely to show up in 5.2), and the reference-tracking mechanism it is meant to support has not yet been seen. But simply finding and converting all callers is expected to be a lengthy process, so the plan is to put the API in place first. 

One significant caller is the block layer. Filesystems hand pages to the block layer (inside BIOs) with a request to perform I/O on those pages. When that I/O completes at some future time, it is the block layer's job to release the references to those pages that were created when the BIO is built. The context in which this happens is far removed from when the BIO was created, so any information needed by the block layer to release these references properly must be stored in the BIO itself. That, of course, is the purpose of this new tracking mechanism: it's all there so that the block layer knows whether to call `put_page()` or `put_user_page()`. 

All of this mechanism only solves one piece of the puzzle: knowing whether a given page has references created by `get_user_pages()` or not. Among the nagging little details that have not yet been addressed is this one: what will filesystems actually do with that information once it's reliably available? There has been talk of using bounce buffers for I/O or simply keeping those pages in a permanently dirty state, but no code has been posted yet. Until that happens and gives developers a look at how this information will be used, it may prove hard to get this new tracking mechanism upstream. Indeed, Glisse indicated in the posting that he does not expect to see it merged before 5.3. One might well expect, though, that there will be some lively discussions about it at the upcoming [Linux Storage, Filesystem, and Memory-Management Summit](<https://events.linuxfoundation.org/events/linux-storage-filesystem-mm-summit-2019/>).
