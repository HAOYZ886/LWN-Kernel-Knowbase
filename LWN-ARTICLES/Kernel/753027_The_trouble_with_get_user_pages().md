---
title: The trouble with get_user_pages()
url: https://lwn.net/Articles/753027/
date: "April 30, 2018"
category: "Memory management-get user pages"
author: "By Jonathan Corbet April 30, 2018 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 30, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

When kernel code needs to work directly with user-space pages, it often calls [`get_user_pages()`](<https://elixir.bootlin.com/linux/latest/source/mm/gup.c#L1067>) (or one of several variants) to fault those pages into RAM and pin them there. This function is not entirely easy to use, though, and recent changes have made it harder to use safely. Jan Kara and Dan Williams led a plenary session at the 2018 Linux Storage, Filesystem, and Memory-Management Summit to discuss potential solutions, but it is not entirely clear that any were found. 

Kara started by saying that he just spent half a year chasing down reports of kernel crashes; now that he has found the reason, he's not sure what to do about it. It comes down to how `get_user_pages()` is used. When it is called, it will translate user-space virtual addresses to physical addresses and ensure that the pages are in memory. Typically the caller will then perform some sort of I/O on those pages. There are a number of [![\[Jan
Kara\]](https://static.lwn.net/images/conf/2018/lsfmm/JanKara-sm.jpg)](<https://lwn.net/Articles/753034/>) mechanisms by which this is done, but it all comes down to passing the addresses of the pages to the devices. When the I/O is complete, the kernel calls [`set_page_dirty()`](<https://elixir.bootlin.com/linux/latest/source/mm/page-writeback.c#L2531>) to mark the pages as dirty and releases its references to the pages. 

Problems can arise when the kernel decides to perform writeback on some of the pages brought in with `get_user_pages()`. The writeback process will write-protect the pages so that user-space cannot modify them until writeback is complete, but it knows nothing about DMA operations started by the driver that called `get_user_pages()`; that I/O may still be ongoing. One failure mode comes about as the result of the filesystem not knowing that pages are changing underneath it; that can lead to crashes in the filesystem code. 

Other crashes can come about if page reclaim removes buffers from the pages before the driver marks them dirty. Problems can result from modification of the data contained in pages while they are under writeback; it is essentially the [stable pages problem](<https://lwn.net/Articles/442355/>) all over again. And there are various data loss or corruption problems associated with use of `fallocate()` on pages that are under I/O — `fallocate()` may want to shuffle pages around, but an ongoing DMA operation will do the wrong thing if that happens. 

Things get even worse if DAX is in use, since the pages in question exist on the storage media itself. If, for example, pages are truncated from a file before DMA completes, the result can be data and metadata corruption. Running DMA directly against blocks that the filesystem is manipulating is hazardous; the filesystem cannot see the elevated reference counts that would indicate that something else is going on with those pages. 

Boaz Harrosh suggested simply preventing writeback on pages with elevated reference counts, but that would be likely to create all kinds of strange side effects. The fact that subsystems like RDMA can hold references on [![\[Dan Williams\]](https://static.lwn.net/images/conf/2018/lsfmm/DanWilliams-sm.jpg)](<https://lwn.net/Articles/753068/>) pages for hours at a time exacerbates this kind of problem. (The group circled for a while on the topic of whether this kind of long-term reference makes sense, without any sort of useful outcome). 

Williams said that the core of the problem is finding a way to allow the kernel to work with pages that have been pinned with `get_user_pages()`. He proposed a set of changes, starting with storing information about pinned pages in the inode (Al Viro was quick to ask: "which inode?") and requiring `get_user_pages()` users to provide a `revoke()` callback. Jérôme Glisse insisted, though, that any call site that could implement `revoke()` could also just use MMU notifiers to detect changes. Williams said that `revoke()` would really just wait for the I/O to complete so that the pages could be released, but Glisse pointed out that, with various types of I/O (such as a camera device streaming video images) the I/O is never really done. There would be no avoiding taking action to stop I/O in such cases. 

Going further, Glisse stated that MMU notifiers are the interface that the kernel has now for dealing with memory-management events. They are called for all page-table entry changes, including write protection; they should be used, he said, rather than reinventing the interface somewhere else. Kara acknowledged that the idea sounds interesting for short-term users of `get_user_pages()`, at least. As the session ran out of time, Glisse said that long-term users could make it work too; the Mellanox RDMA driver "did it right", for example. Of course, he acknowledged, the fact that this interface has its own memory-management unit helps. The kernel should, he said, "be mean" to hardware that lacks such capabilities. 

About the only hard conclusion from this discussion was that more discussions are needed before the developers will get a real handle on this problem.
