---
title: The state of the page in 2025
url: https://lwn.net/Articles/1015320/
date: "March 26, 2025"
category: "Memory management-Folios"
author: "By Jonathan Corbet March 26, 2025 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 26, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The [folio transition](<https://lwn.net/Articles/849538/>) is one of the most fundamental kernel changes ever made; it can be thought of as being similar to replacing the foundation of a building while it remains open for business. So it is not surprising that, for some years, the annual Linux Storage, Filesystem, Memory-Management, and BPF Summit has included a session on the state of this transition. The 2025 Summit was no exception, with Matthew Wilcox updating the group on what has been accomplished, what remains to be done, and where some of the significant problems are. 

The initial idea behind folios, Wilcox began, was to manage pages in larger blocks; the experience of the last few years shows that it works. Later on, the goal of shrinking the `page` structure, which represents a single page in memory was added. Even later came objectives like enabling filesystems with block sizes larger than the page size and improving the debugability and clarity of the memory-management subsystem. There has been the accumulation of a lot of cruft in that subsystem over the years, he said; the folio transition is an opportunity to clean some of it out. 

[![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2025/lsfmm/MatthewWilcox-sm.png)](<https://lwn.net/Articles/1015332/>) There are two understandings of what a folio is. The first, which he called the "Ottawa interpretation", is what he intended initially; it was, in essence, just the head page of a compound page. Over time, though, the conception of folios has shifted toward the "New York interpretation", much of which is the work of Johannes Weiner. In that view, folios are an opportunity to shrink `struct page` to a single `u64` memory descriptor. Progress is being made toward that goal, but it will not be achieved this year. 

Since a folio is an independent structure, it can grow as needed. The size of `struct page`, instead, is strictly constrained; since there is one per page, it must be as small as possible. Even though `struct folio` is getting larger, there will be a lot fewer of them, so the overall memory-management overhead will decrease. Getting to the point where `struct page` can be replaced will require quite a bit of work, still. 

In 2025, the objective is to get to the point where `struct folio` is indeed a separate structure from `struct page` and can be allocated independently. Then, data can be removed from `struct page`, shrinking it, but not yet all the way. 

Wilcox noted that he is getting tired of converting filesystems to folios, which is a necessary step on the path to a smaller `page` structure. So he is considering adding a new kernel configuration option, `CONFIG_SEPARATE_FOLIO`, that would compile out any code that is not yet prepared for a separate `folio` structure. That would allow the creation of a kernel where the separate-folio changes can be tested, even if it isn't yet a kernel that supports all of the important features (networking, say) that users might actually want. 

The upcoming work involves removing all references to a number of `struct page` fields, including `mapping`, `index`, and `lru`. The networking subsystem's pagepool mechanism will need to be separated out. There is also the `page` pointer in `struct folio`, which points to the underlying `page` structure, and a lot of ""more interesting"" casts scattered through the kernel code that perform a similar function; those will have to be fixed. For example, the `buffer_head` structure has page and folio pointers in a single union and will need to be fixed. Another near-future change will be adapting the slab allocator to use the separate [`slab` structure](<https://lwn.net/Articles/871982/>). 

Wilcox then reviewed some of the goals he had covered in [the 2024 update](<https://lwn.net/Articles/973565/>). Many of them remained incomplete; in his defense, he said, it had only been ten months since last year's conference. Given those two months — essentially one whole development cycle — he would have checked off more of them. 

Some things were definitely accomplished, though. The [`zpdesc` memory descriptor](<https://lwn.net/Articles/1002362/>) was added to replace `struct page` use in the zswap subsystem. It is currently an overlay on `struct folio`, but Wilcox thinks it could be made more space-efficient. The [ability to use a filesystem block size larger than the page size](<https://lwn.net/Articles/1009548/>) has been ""quite the journey"", but that ability now exists for the XFS and bcachefs filesystems. It should be easy to add to other filesystems as well — at least, once those filesystems are able to support large folios. 

Another achievement is the ability to allocate and free [frozen pages](<https://lwn.net/Articles/1000654/>), which have no reference count. The adoption of this feature, he said, is ""borrowing pain from the future"". Recently, the network stack [found this pain](<https://lwn.net/Articles/1013408/>); happily, this problem turned up and was fixed before the 6.14 release. Wilcox would like to see more use of frozen pages in general, but he pointed out that there will always be some places where reference counts will be needed. 

Another important step forward is [imprecise mapping-count tracking for large folios](<https://lwn.net/Articles/1013649/>), which changes the kernel to track the number of processes mapping a folio, rather than the number of mappings, once the number of processes exceeds two. This work enables precise tracking for the common cases while maintaining correctness in the more exotic cases. 

It is now possible to create large folios in [`generic_perform_write()`](<https://elixir.bootlin.com/linux/v6.13.7/source/mm/filemap.c#L4007>), which, he said, is a big deal; it can double write performance in some tests. That result wasn't surprising, he said, since using large folios frees the kernel from having to manage large numbers of base pages. Meanwhile, the `bh_page` pointer in buffer heads is now unused, all that remains is to actually delete it. There has also been a lot of work removing the wrapper functions around the various page flags, further reducing the role of `struct page`. 

There is currently some use of page types, which will eventually be stored in the memory descriptor. The type is stored in the `page` structure, but it is overlaid by the `mapcount` field, so it cannot be used in mapped pages. There is some trickery being used to distinguish mapping counts (positive numbers) from the page types, which are indicated by a negative mapping count. Various types of pages, including hugetlb, slab, zsmalloc, [unaccepted](<https://lwn.net/Articles/928328/>), and large-kmalloc pages, are identified by page types now. 

[Page flags](<https://lwn.net/Articles/974515/>) have long been in short supply; there is exactly one of them available at the moment. The `PG_slab` flag is gone now, as is `PG_error`, which turned out to just be the inverse of `PG_uptodate`. Those changes freed two flags, but then [`PG_dropbehind`](<https://git.kernel.org/linus/cceba6f7e46c>) (which might be renamed `PG_reclaim`) was added. The `PG_uncached` flag is now an alias for `PG_arch_2`, while `PG_mappedtodisk` overlays `PG_owner_2`. The `PG_private_2` flag is almost unused, but the Ceph and NFS filesystems still need it. `PG_private` needs more work before it can be removed, since it is used for a lot of different things in different places. Often it is used to indicate that there is something stored in the `private` field of `struct page`, but a test for `NULL` should be usable instead (some members of the group disagreed with that claim). Most of the existing page flags will eventually become folio flags, he said, while `PG_hwpoison` will become a page type. 

Wilcox concluded with some suggestions for anybody who wants to help with the folio transition. At the top of his list was working to make more filesystems support large folios; that, he said, is good for the system as a whole. The "bitmap" MD target needs to stop using buffer heads, but he hasn't looked at MD in many years and is afraid of it. Removing any uses of the `page` member of `struct folio` and the `lru` page field would also be useful. 

There have been many developers who have helped with this work so far, and he thanked them all. It has been a fun project; he was looking forward, he said in jest, to next year's Summit where he will be able to say that it is complete.
