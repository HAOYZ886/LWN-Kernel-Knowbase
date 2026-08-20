---
title: Huge pages in the ext4 filesystem
url: https://lwn.net/Articles/718102/
date: "March 28, 2017"
category: "Filesystems-ext4; Huge pages"
author: "By Jonathan Corbet March 28, 2017 LSFMM 2017"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 28, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

When the transparent huge page feature was added to the kernel, it only supported anonymous (non-file-backed) memory. In 2016, [support for huge pages in the page cache](<https://lwn.net/Articles/686690/>) was added, but only the tmpfs filesystem was supported. There is interest in expanding support to other filesystems, since, for some workloads, the performance improvement can be significant. Kirill Shutemov led the only session that combined just the filesystem and memory-management tracks at the 2017 Linux Storage, Filesystem, and Memory-Management Summit in a discussion of adding huge-page support to the ext4 filesystem. 

He started by saying that the tmpfs support works well now, so it's time to take the next step and support a real filesystem. [Compound pages](<https://lwn.net/Articles/619514/>) are used to represent huge pages in the system memory map; the first of the range of (small) pages that makes up a huge page is the head page, while the rest are tail pages. Most of the important metadata is stored in the head page. Using compound pages allows the entire huge page to be represented by a single entry in the least-recently-used (LRU) lists, and all buffer-head structures, if any, are tied to the head page. Unlike DAX, he said, transparent huge pages do not force any constraints on a file's on-disk layout. 

With tmpfs, he said, the creation of a huge page causes the addition of 512 (single-page) entries to the radix tree; this cannot work in ext4. It is also necessary to add DAX support and to make it work consistently. There [![\[Kirill Shutemov\]](https://static.lwn.net/images/conf/2017/lsfmm/KirillShutemov-sm.jpg)](<https://lwn.net/Articles/718103/>) are a few other problems; for example, readahead doesn't currently work with huge pages. The maximum size of the readahead window is 128KB, far less than the size of a huge page. He was not sure if that was a big deal or not but, if it is, it will need to be fixed. Huge pages also cause any [shadow entries](<https://lwn.net/Articles/495543/>) in the page cache to be ignored, which could worsen the system's page-reclaim decisions. 

He emphasized that huge pages need to avoid breaking existing semantics. That means that it will be necessary to fall back to small pages at times. Page migration was one example of when that can happen. A related problem is that a lot of system calls provide 4KB resolution, and that can interfere with huge-page use. Use of encryption in ext4 will also force a fallback to small pages. 

Given all that, he asked, is there any reason not to pursue the addition of huge-page support to ext4? He has patches that have been circulating for a while; his current plan is to rebase them onto the current [page cache work](<https://lwn.net/Articles/712467/>) and repost them. 

Jan Kara asked if there was a need to push knowledge of huge pages into every filesystem, adding complexity, or if it might be possible for filesystems to always work with small pages. Shutemov responded that this is not always an option. There is, for example, a single up-to-date flag for the entire compound page. It makes sense to work to make the abstractions cleaner and hide the differences whenever possible, and he has been doing that, but the solution is not always obvious. 

Kara continued, saying that there needs to be some sort of proper data structure for tracking sub-page state. The kernel currently uses a list of buffer-head structures, but that could perhaps be changed. There might be an advantage to finer-grained tracking. But he repeated that he doesn't see a reason why filesystems should need to know about the size of pages as stored in the page cache, and that teaching every filesystem about a variably sized page cache will be a significant effort. Shutemov agreed with the concern, but said that the right approach is to create an implementation for a single filesystem, get it working, then try to create abstractions from there. 

Matthew Wilcox, instead, complained that the current work only supports two page sizes, while he would like it to handle any compound page size. Generalizing the code to make that possible, he said, would make the whole thing cleaner. The code doesn't have to actually handle every size from the outset, but it should be prepared for that. 

Trond Myklebust said that he would like to have proper support for huge pages in the page cache. In the NFS code, he has to do a lot of looping and gathering to get up to reasonable block sizes. Ted Ts'o asked whether the time had come to split the notion of a page's size (`PAGE_SIZE`) and the size of data stored in the page cache (`PAGE_CACHE_SIZE`). The kernel used to treat the two differently, but that distinction was removed some time ago, resulting in cleaner code. Wilcox responded that the meaning of `PAGE_CACHE_SIZE` was never well defined in the past, and that generalizing the handling of page-cache size is not a cleanup, it's a performance win. He suggested it might also make it easier to support multiple block sizes in ext4, though Shutemov was quick to add that he couldn't promise that. 

The problem with larger block sizes, Ts'o said, comes about when a process takes a fault on a 4KB page, and the filesystem needs to bring in a larger block. This has never been easy. The filesystem people say it's a memory-management problem, while the memory-management people point their finger at filesystems. This situation has stayed this way for a long time, he said. Wilcox said he wants it to be a memory-management problem; his work to support variable-sized pages in the page cache should address much of it. 

Andrea Arcangeli said that the real problem happens when larger pages are not available for allocation. The transparent huge pages code is careful to never require such allocations; it will always fall back to smaller pages. He would not like to see that change. Instead, he said, the real solution is to increase the base page size. Rik van Riel answered that, if the page cache contains more large pages, they will be available for reclaim and should be easier to allocate than they are now. 

As the session closed, Ts'o observed that the required changes are much larger on the memory-management side than on the ext4 side. If the group is happy with this work, perhaps it's time to merge it with the idea that the remaining issues can be fixed up later. Or, perhaps, it's better to try to further evolve the interfaces first. It is, he said, more of a memory-management decision, so he will defer to that group. Shutemov said that the page-cache interface is the hardest part; he will look at making the interface with filesystems cleaner. But, he warned, it doesn't make sense to try to abstract everything from the outset.
