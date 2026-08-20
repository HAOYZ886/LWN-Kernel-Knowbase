---
title: "A memory-folio update"
url: https://lwn.net/Articles/893512/
date: "May 4, 2022"
category: "Memory management-Folios"
author: "By Jonathan Corbet May 4, 2022 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 4, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

The [folio project](<https://lwn.net/Articles/849538/>) is not yet two years old, but it has already resulted in significant changes to the kernel's memory-management and filesystem layers. While much work has been done, quite a bit remains. In the opening plenary session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>), Matthew Wilcox provided an update on the folio transition and led a discussion on the work that remains to be done. 

Wilcox began with an overview of the folio work, a more complete description of which can be found in the above-linked article. In short, a folio is a way of representing a set of physically contiguous base pages. It is a response to a longstanding confusion in the memory-management subsystem, wherein a "page" can refer either to a base page or a larger [compound page](<https://lwn.net/Articles/619514/>). Adding a new term disambiguates the term "page" and simplifies many memory-management interfaces. 

Beyond terminology, there is another motivation for the folio work. The kernel really needs to manage memory in larger chunks than 4KB base pages. There are millions of those pages even on a typical laptop; that is a lot of pages to manage and a pain to deal with in general, causing the waste of a lot of time and energy. Better interfaces are needed to facilitate management of larger units, though; folios are meant to be that better interface. 

#### Current status

A folio is represented by [`struct folio`](<https://elixir.bootlin.com/linux/v5.18-rc5/source/include/linux/mm_types.h#L226>); it is essentially an alias for the head page of a compound page. Wilcox has been adding uses of folios into the kernel over the course of the last year; this project has come a long way but is not yet complete. 

One open question concerns when the kernel should allocate large folios — those containing more than one base page. Only the readahead code allocates them now; the filesystem write path still does everything in terms of base [![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2022/lsfmm/MatthewWilcox-sm.png)](<https://lwn.net/Articles/893613/>) pages. If writes are done to large folios that were brought in via readahead, they will see and use those large folios. Appending to a file will always use base pages, though. There are almost certainly advantages to using large folios in the write path, but it will be necessary to figure out what the criteria for creating them will be. 

Meanwhile, the process of converting filesystem code to folios continues. Wilcox encouraged filesystem developers to look for infrastructure that already exists when possible rather than reimplementing it themselves. He pointed out the support layer for network filesystems that was recently rewritten by David Howells. It would also be good for filesystems to move away from the old buffer-head APIs and use the relatively new iomap infrastructure whenever possible. 

Ted Ts'o said that more guidance on conversion to iomap would be useful. Moving a filesystem over can be a daunting task, he said, but developers should understand that it can be done incrementally. For example, a filesystem's read path can be converted while leaving the write path unchanged for now. This can be useful, Wilcox agreed, especially since iomap is still missing some capabilities, such a support for features like fs-verity or compression. That lack is often more problematic on the write side than on the read side. 

#### API complaints

Josef Bacik said that one particularly annoying problem for Btrfs is that the memory-management subsystem's page locks must be taken before filesystem-level locks. That makes it hard at the filesystem level, and gets in the way of needed features like range locking. He would love to see this issue addressed, but knows that it will not be easy. Wilcox admitted that this problem had not been on his radar at all, but it is something he will have to look into. Chris Mason noted that the problem is not specific to Btrfs; other filesystems have encountered similar difficulties over the years. 

Bacik also said that page reclaim driven by memory management can also be problematic, and the interface to filesystems is not great. It would be good, he said, to be able to distinguish requests like "please free whatever memory you can now" from requests to free specific pages. Wilcox said that much of the kernel's reclaim machinery may not be relevant anymore; it was designed in the days when filesystems were far less capable than they are now. Good filesystems now are already keeping all of their drives busy doing writeback; there is really little more that they can do if the memory-management code wants them to free specific pages. Perhaps the memory-management subsystem should simply stop requesting the reclaim of pages that reach the end of the least-recently-used (LRU) list, he suggested. 

There is a possible way to test that idea, he said; perhaps filesystems should simply remove their implementation of the `writepage()` [address-space operation](<https://elixir.bootlin.com/linux/v5.18-rc5/source/include/linux/fs.h#L340>). Howells said that he had done that in the AFS filesystem, with seemingly good results. Some other filesystems, including 9P, will be harder though. 

The problem there, Ts'o said, is that the memory-management subsystem is trying to solve multiple problems at the same time. When responding to global memory pressure, it just needs some pages to be freed and will not be that picky about where they happen to be. Once control groups enter the picture, though, it becomes necessary to relieve memory pressure within a specific container; that requires reclaim to be more focused. When compaction is being performed to create huge pages, it comes down to freeing specific pages. These cases need to be thought about separately. Removing `writepage()` may help with the global problem, but the need to free specific pages doesn't go away. 

Wilcox expressed a hope that widespread use of large folios will help with the compaction problem at least, since there should be far less fragmentation in the first place. In some benchmark runs he has seen the length of the LRU lists reduced by a factor of 1000, which is "just insane". 

On the other hand, he said, one potential problem resulting from large folios may be a form of write amplification. Dirty state is tracked at the folio level, not at the level of the individual base pages contained therein; when the time comes to write out data, the entire folio will be written even if only one byte has changed. This will increase the write bandwidth used by the system, but should also help to reduce fragmentation on copy-on-write filesystems. He said that he didn't expect "serious trouble" though. 

Others were not so sure. Mason pointed out that Jens Axboe has been putting in a considerable amount of effort to make it easy to perform small operations in io_uring. This work is specifically motivated by write-bandwidth concerns. Axboe added that bandwidth is indeed a concern, but is more of a problem on the read side than with writes. There was some discussion on how big the problem actually is; one developer pointed out that the situation will vary depending on the filesystem in use. For a network filesystem with high latency, writing too much data may be better than doing multiple round trips with the server. There was a general agreement that better metrics are needed to understand the situation properly. 

#### Longer-term goals

Moving on, Wilcox said that he is still in the process of converting the address-space operations provided by filesystems to folios; there are still a couple of them to be done. In many cases, this "conversion" is a matter of changing a function prototype to accept a pointer to `struct folio` rather than to `struct page`, then adding a line like: 

```
struct page *p = (struct page *) folio;
```

This pattern is, he said, "a bad code smell"; it is a sign that the code in question needs further work. The plan is to eventually convert every filesystem to folios — but not necessarily to the point of using _large_ folios. 

There is an underlying motivation behind this work: he hopes to eventually remove one of the big union members from `struct page`, once filesystems are no longer using that structure. Memory-management developers, he said, want to put a lot more information into `struct page`, but there are strong reasons to not make that structure any larger. So, instead, he would like to shrink it; perhaps, someday, it can be reduced (from 64 bytes) to a single pointer. Even better, that could be one pointer per folio, rather than one structure per page, allowing the kernel to get back the 1.6% of memory that is currently used to hold `page` structures. 

That, he said, will allow companies to save money on memory and use it to send their developers to more conferences. 

Howells said that it would be good to eventually get rid of the `write_begin()` and `write_end()` address-space operations; Wilcox agreed, saying that they were originally designed for the needs of ext3, and later filesystems have had to fit into that model. Goldwyn Rodrigues pointed out that iomap is not currently using those callbacks. 

Kent Overstreet complained about the practice of passing around structures full of callbacks, which he described as an "old model" of API design. Bacik said, though, that he doesn't really care about the API as long as it lets him focus on Btrfs and not have to worry about how memory management works. Wilcox answered that much of his work has been aimed at making filesystems easier to write in general, and he hopes that folios help in that regard. Nothing in filesystems should have to care about pages, he said, except for, possibly, the page-fault path. 

Overstreet, though, objected that developers should care more about such things. Many of the kernel's internal interfaces have aged badly; developers should be talking about what the pain points are and how to remove them. Bacik said that the kernel needs developers who care about these interfaces specifically; he, personally, is on the edge of burnout and can't take on other tasks. So he is happy about the folio work; there is an owner who cares about the interface and is working to make it better. He said that this is hard, thankless work, and thanked Wilcox for taking it on. 

Wilcox closed the session by acknowledging that the folio work is imposing costs on many other developers, and said that he feels the weight of that cost. Developers have made the costs clear to him, some more politely than others. He thanked Bacik for his comments, saying the he is glad that somebody, at least, sees the benefit of this work.
