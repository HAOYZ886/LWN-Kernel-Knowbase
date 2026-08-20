---
title: Multipage bio_vecs
url: https://lwn.net/Articles/685928/
date: "May 4, 2016"
category: Block layer
author: "By Jake Edge May 4, 2016 LSFMM 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
May 4, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

In the block layer, larger I/O operations tend to be more efficient, but current kernels limit how large those operations can be. The `bio_vec` structure, which describes the buffer for an I/O operation, can only store a single page of tuples (of page, offset, and length) to describe the I/O buffer. There have been efforts over the years to allow multiple pages of array entries, so that even larger I/O operations can be held in a single `bio_vec`. Ming Lei led a session at the 2016 Linux Storage, Filesystem, and Memory-Management Summit to discuss [patches](<https://lwn.net/Articles/683856/>) to support `bio_vec` structures with multiple pages for the arrays. 

Multipage `bio_vec` structures would consist of multiple, physically contiguous pages that could hold a larger array. It is the correct thing to do, Lei said. It will save memory as there will be fewer `bio_vec` structures needed and it will increase the transfer size [ ![\[Ming Lei\]](https://static.lwn.net/images/2016/lsf-lei-sm.jpg) ](<https://lwn.net/Articles/685989/>) for each `struct bio` (which contains a pointer to a `bio_vec`). Currently, the single-page nature of a `bio_vec` means that only one megabyte of I/O can be contained in a single `bio_vec`; adding support for multiple pages will remove that limit. 

Jens Axboe agreed that there are benefits to larger `bio_vec` arrays, but was concerned about requesters getting physically contiguous pages. That would have to be done when the `bio` is created. Lei said that it is not hard to figure out how many pages will be needed before creating the `bio`, though. 

All of the "magic" is in the `bio_vec` and `bio` iterators, one developer in the audience said. So there would be a need to introduce new helpers to iterate over the multipage `bio_vec`. The new name for the helper would require that all callers change, which would provide a good opportunity to review all of the users of the helpers, Christoph Hellwig said. 

The patches also clean up direct access to some fields in `bio` structures: `bi_vcnt`, which tracks the number of entries in the `bio_vec` array, and the pointer to the `bio_vec` itself (`bi_io_vec`). 

Axboe was concerned about handling all of the different special cases. There need to be "some real wins" in the patch set, since the memory savings are not all that huge. He is "not completely sold on why multipage is needed". 

Hellwig agreed that the memory savings were not particularly significant, but that there is CPU time wasted in iterating over the segments. At various levels of the storage stack, the kernel has to iterate over the `bio` and `bio_vec` structures that make up I/O requests, so consolidating that information will save CPU time. There are also many needed cleanups in the patches, he said, so those should be picked up; "then, hopefully, we can get to the multipage `bio_vec`s". 

Axboe said that the patches have been posted, but are not all destined for 4.7. He will queue up some of the preparatory patches, but the rest "need some time to digest".
