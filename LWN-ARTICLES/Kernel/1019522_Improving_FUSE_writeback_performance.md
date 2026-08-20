---
title: Improving FUSE writeback performance
url: https://lwn.net/Articles/1019522/
date: "May 6, 2025"
category: "Filesystems-In user space; Memory management-Writeback"
author: "By Jake Edge May 6, 2025 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
May 6, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

In a combined filesystem and memory-management session at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF), Joanne Koong led a discussion on improving the writeback performance for the [Filesystem in Userspace](<https://github.com/libfuse/libfuse?tab=readme-ov-file#libfuse>) (FUSE) layer. Writeback is how data that is written to the filesystem is actually flushed to the disk; it is the process of writing dirty pages from the page cache to storage. The current FUSE implementation allocates unmovable memory, then copies the dirty data to it before initiating writeback, which is slow; Koong wanted to change that behavior. Since the session, she has [posted a patch set](<https://lwn.net/ml/all/20250414222210.3995795-1-joannelkoong@gmail.com/>) that has been [applied](<https://lwn.net/ml/all/CAJfpegveOFoL-XzDKQZZ4U6UF_AetNwTUDbfmf7rdJasRFm3xA@mail.gmail.com/>) by FUSE maintainer Miklos Szeredi. 

Koong started the session with a description of the current FUSE writeback operation. A temporary page is allocated in the unmovable memory zone for each dirty page and the data is copied to the temporary page. After that, writeback is initiated on the temporary pages and the original pages can immediately have their writeback state cleared. That extra allocation and copying work is expensive, but is needed so that the pages do not move while the writeback operation is underway. 

Benchmarks have shown around 45% improvement in throughput for writes without the temporary pages, she said. Beyond that, eliminating the copy simplifies the internals of FUSE. There is currently a red-black tree tracking the temporary pages that could be eliminated. It also makes the conversion of FUSE to use large folios much cleaner. 

Back in November, she sent a [proposed solution](<https://lwn.net/ml/all/20241122232359.429647-1-joannelkoong%40gmail.com/>) that removes the temporary pages, which means that the writeback state will not be cleared immediately anymore. In order to avoid deadlocks, the patch set added a new mapping flag `AS_WRITEBACK_INDETERMINATE` that filesystems can set on inode mappings to say that writeback may take an indeterminate amount of time to complete; FUSE will set the flag on its mappings, which can be used to avoid deadlocks in the writeback machinery. 

[ ![\[Joanne Koong\]](https://static.lwn.net/images/2025/lsfmb-koong-sm.png) ](<https://lwn.net/Articles/1020167/>)

That patch set was rejected, Koong said, primarily because it would allow buggy or malicious FUSE servers to hold up migration indefinitely by not ever completing the writeback of some pages. That would increase memory fragmentation and thwart attempts to allocate contiguous memory. Allocating the temporary pages can also fragment memory, but those are made in unmovable memory, which is less problematic to fragment than movable memory. Other parts of FUSE already have this problem, including readahead and writethrough splicing (using [`splice()`](<https://www.man7.org/linux/man-pages/man2/splice.2.html>)), but ""we shouldn't try to add more of this, we should try to eliminate it if we can"". 

Several options were discussed in the thread, but the most promising idea, providing a mechanism to cancel writeback if pages need to migrate, does not work. The problem is that pages can be spliced, she said, and the writeback cannot be canceled for those pages. Another viable possibility is to have a dedicated area in the movable zone for pages that may be unmovable for indeterminate amounts of time. That would reduce the impact of the fragmentation to only that area of memory. Alternatively, unprivileged FUSE servers that behave badly, by not completing writeback in a timely fashion or by having too many pages under writeback, could just be killed. 

David Hildenbrand said that there was some discussion of disallowing splicing for unprivileged FUSE servers; ""you're not trustworthy enough to let you do that"". That would allow canceling writeback, but Koong was not sure that was the right path forward. What followed was some fast-moving, hard-to-follow discussion on various possibilities for avoiding the edge cases that can lead to deadlock. 

Omar Sandoval asked about the feasibility of just killing the misbehaving unprivileged servers as was suggested. Koong said that it was a reasonable solution, though it may not be backward compatible because existing servers are not expecting it. But she thinks that something along those lines should already have been done as a protection mechanism. 

Sandoval asked what a reasonable timeout value should be. There is a balance to be struck; ""if you're a FUSE server and you've gone out to lunch for 30 minutes, I don't care about your backwards compatibility, you already broke everything"". Hildenbrand said that is a difficult problem to solve; any timeout chosen will sometimes be too large or too small. Sometimes the data will be valuable enough that a long wait is acceptable, but, say, 30 seconds may already be too long to hold off an allocation. 

It would be his wish to find some easy way to handle the common cases where the pages can just be migrated, which might mean prohibiting the use of `splice()`. He wondered what the implication of that prohibition would be. Koong said that the FUSE servers could be audited for the use of `splice()` and the problem could be discussed with the developers. Josef Bacik said that the kernel could just fall back to doing an internal copy when `splice()` is requested from an unprivileged server. 

The crux of the problem seems to be the unmovable nature of the memory that is under writeback, he continued; if some new way could be found to use movable memory without doing a copy, that would be ideal. ""We love `splice()` because it's faster, but it sounds like we need to invent a new zero-copy mechanism that uses movable memory"". 

The ability to mount FUSE filesystems as an unprivileged user makes them so problematic, Jeff Layton said; any random user can start a server that can grab a bunch of memory and not handle it properly. That is what the system needs to guard against; doing so with ""draconian measures"" like killing the server is not unreasonable. He suggested finding a way to maintain compatibility with the existing servers and to provide a zero-copy mechanism for new ones; in his mind, it is not out of the question to rewrite some of the old FUSE servers to take advantage of newer features. Koong agreed, but said that it would be Szeredi's call on what should be done in that regard; she was not clear on what his thinking is.
