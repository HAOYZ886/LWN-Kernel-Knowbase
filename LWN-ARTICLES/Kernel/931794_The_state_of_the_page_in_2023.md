---
title: The state of the page in 2023
url: https://lwn.net/Articles/931794/
date: "May 17, 2023"
category: "Memory management-Folios"
author: "By Jonathan Corbet May 17, 2023 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 17, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The conversion of the kernel's memory-management subsystem over to [folios](<https://lwn.net/Articles/849538/>) was never going to be done in a day. At a plenary session at the start of the second day of the 2023 [Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Matthew Wilcox discussed the current state and future direction of this work. Quite a lot of progress has been made — and a lot of work remains to be done. 

There is no single best page size for the management of memory in a Linux system, Wilcox began. On some benchmarks, using 64KB pages produces significantly better results, but others do better with 4KB base pages. In general, though, managing a system with 4KB pages is inefficient at best; at that size, the kernel must scan through millions of `page` structures to provide basic management functions. Beyond that, the `page` structure is badly overloaded and difficult to understand. If it needs to grow for one page type, it must grow for all, meaning in practice that it simply cannot grow, because somebody will always push back on it. 

[![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2023/lsfmm/MatthewWilcox-sm.png)](<https://lwn.net/Articles/931795/>) To address this problem, Wilcox and others are trying to split `struct page` into a set of more specialized structures. Eventually `struct page` itself will shrink to a single pointer, where the least-significant bits are used to indicate what type of usage-specific structure is pointed to. The work to [move slab-allocator information](<https://lwn.net/Articles/871982/>) out of `struct page` has already been completed. There are plans (in varying states of completion) to make similar changes for pages representing page tables, compressed swap storage, [poisoned](<https://lwn.net/Articles/348886/>) pages, folios, free memory, device memory, and more. 

When will this work be done? Wilcox waved his hands and said "two years" to general laughter. There have been 1,087 commits in the mainline so far that mentioned folios. The page cache has been fully converted, as have the slab allocators. The tail-page portion of [compound pages](<https://lwn.net/Articles/619514/>) has been moved to folios, allowing the removal of another section from `struct page`. The [`address_space_operations` structure](<https://elixir.bootlin.com/linux/v6.4-rc1/source/include/linux/fs.h#L375>) has been converted — except for three functions that will soon be deleted entirely. 

There are three filesystems (XFS, AFS, and EROFS) that have been fully converted, as have the iomap and netfs infrastructure layers. A number of other filesystems, including ext4, NFS, and tmpfs, can use single-page folios now. The `get_user_pages()` family of functions uses folios internally, though its API is still based on `struct page`. Much of the internal memory-management code has been converted. One might be tempted to think that this work is nearly done, but there is still a lot of code outside of the memory-management layer that uses `struct page` and will need to be converted. 

Every conversion that is done makes the kernel a little smaller, Wilcox said, due to the simplifying assumption that there are no pointers to tail pages. Over time, this shrinkage adds up. 

There are plenty of topics to discuss for the future, he said. One is the conversion of the buffer-head layer, which is in progress (and which was the subject of the next session). Folios will make it easier to support large filesystem block sizes. The `get_user_pages()` interfaces need to be redesigned, and there are more filesystem conversions to do. A big task is enabling multi-page anonymous-memory folios. Most of the work done so far has been with file-backed pages, but anonymous memory is also important. 

One change that is worth thinking about, he said, is reclaiming the `__GFP_COMP` allocation flag. This flag requests the creation of a compound page (as opposed to a simple higher-order page); that results in the addition of a bunch of metadata to the tail pages. This is useful for developers working on kernel-hardening projects, who can use it to determine if a copy operation is overrunning the underlying allocation. They would like the kernel to always create compound pages and simply drop non-compound allocations so, Wilcox suggested, the page allocator could just do that by default and drop the `__GFP_COMP` flag entirely. 

He mentioned some pitfalls that developers working on folio conversions should be aware of. Some folio functions have different semantics than the page-oriented functions they replace; the return values may be different, for example. These changes have been carefully thought about, he said, and result in better interfaces overall, but they are something to be aware of when working in this area. 

Multi-page folios can also cause surprises for code that is not expecting them. He mentioned filesystems that check for the end of a file by calculating whether an offset lands within a given page; now they must be aware that it could happen anywhere within a multi-page folio. Anytime a developer encounters a definition involving the string `PAGE` (`PAGE_SIZE`, for example), it is time to be careful. And so on. 

There are also, he concluded, a few misconceptions about folios that are worth clearing up. One of those is that there can be only one lock per folio; he confessed that he doesn't quite understand why there is confusion here. There was always just one lock per compound page as well. The page lock is not highly contended; whenever it looks like a page lock is being contended, it is more likely to be an indication of threads waiting for I/O to complete. 

Some developers seem to think that dirtiness can only be tracked at the folio level. It is still entirely possible to track smaller chunks within a folio, though; that is up to the filesystem and how it handles its memory. The idea that page poisoning affects a whole folio is also incorrect; that is a per-page status. 

As the session wound down, David Hildenbrand said that, while folios are good, there is still often a need to allocate memory in 4KB chunks. Page-table pages, he said, would waste a lot of memory if allocated in larger sizes. What is really needed is the ability to allocate in a range of sizes, depending on how the memory will be used. Wilcox closed the session by saying that is exactly the outcome that the developers are working toward.
