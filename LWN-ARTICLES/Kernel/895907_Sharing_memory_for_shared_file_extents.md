---
title: Sharing memory for shared file extents
url: https://lwn.net/Articles/895907/
date: "May 24, 2022"
category: "Filesystems-Copy on write; Memory management-Page cache"
author: "By Jake Edge May 24, 2022 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
May 24, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

On the second day of the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Goldwyn Rodrigues led a combined filesystem and memory-management session on saving memory when reading files that share extents. That kind of sharing can occur with copy-on-write (COW) filesystems, reflinks, snapshots, and other features of that sort. When reading those files, memory is wasted because multiple copies of the same data is stored in the page cache, so he wanted to explore adding a cache specifically to handle that. 

When two files share an extent, their inodes point at the same data blocks on the disk, though they seem to be completely independent files. When those files are read, each gets copied separately into the page cache. That wastes memory, but there are also other costs: reading from the disk, computing checksums, decompressing, and so on. 

His idea is to create a device cache ("not a buffer cache" because that would cause nightmares, he said) within the page cache that would only store a single copy of these pages. His [RFC implementation](<https://lwn.net/ml/linux-fsdevel/cover.1634933121.git.rgoldwyn@suse.com/>) back in October used the inode of the device special file of the underlying device, rather than that of the file in the filesystem, to store the shared extents in the page cache. He described how the cache would work for multiple scenarios (buffered read, buffered write, direct I/O, and `mmap()`), starting with the simplest. 

[ ![\[Goldwyn Rodrigues\]](https://static.lwn.net/images/2022/lsfmm-rodrigues-sm.png) ](<https://lwn.net/Articles/896057/>)

A buffered read would check the page cache for the file and, if the page is not found there, it would calculate the device offset from the read offset in the file and look in the shared-extent cache to see if the page lives there. If not, it would read the data from the disk and add it to the shared-extent cache. Buffered writes would always go to the page cache, because any write ends the sharing of the extent. For writes of a partial page, though, the shared-extent cache would be checked for the rest of the data for that page. 

The harder problem is for direct I/O (DIO), because a shared-extent cache kind of defeats the purpose of DIO, which is to circumvent the page cache. But if the shared-extent cache were used, DIO writes would need to check that cache and remove pages from it since the extents would no longer be shared. But Matthew Wilcox cautioned that even for reads, DIO needs to actually go to the disk because of shared storage, where some other machine may have written to the device. In addition, there are applications that are trying to save CPU cycles and want the DMA from the device to occur; the alternative is to copy the data out of the shared-extent cache using the CPU and "to that application, CPU is more important than bus bandwidth". Rodrigues said that changes in shared storage will require invalidating the caches across the cluster. 

Supporting `mmap()` is "sort of a gray area for me", he said; he is not sure that his way to do so is the right one. There is a read-only mapping for the shared pages and any writes to those pages will result in a page fault that can handle the COW operation. 

He wondered if there should be some kind of differentiation for reads that are targeting shared extents or whether all reads should go through the new cache. Josef Bacik said that he thinks it should just be a new kind of inode that, for Btrfs, maps to its logical byte-number addressing, rather than to anything device-specific. From there it is just treated like any other inode, so, for example, the memory-management (MM) subsystem can ask the filesystem to shrink its inode cache and these cache objects would just be handled normally. 

Beyond that, Bacik does not want to see this as a mount option as it was in the RFC patch set, "death to all mount options"; it should just always work, he said. For Btrfs, he thinks that all reads should go to the new cache because a snapshot could happen at any time. For DIO, the page cache entries should just be invalidated and applications using DIO will not get the benefit of this feature 

There is a question of how the cache gets flushed, since closing the file does not mean that others are not using the pages or won't soon, Rodrigues said. Maybe it makes sense to wait until the inode is evicted. But Bacik said that the starting point should be to not flush these pages at all and let the MM subsystem evict pages as needed. It will not reach the point of an out-of-memory (OOM) condition because the MM will tell the filesystem to invalidate pages before that happens. 

There are some questions, he said, about how to share a single page across multiple mappings for different inodes; how does the system ensure that the COW happens when writes are done and how does the page get reclaimed properly when there are a lot of inodes referencing it? Wilcox said that made for a good opportunity to talk about some plans he has for splitting `struct page` and `struct folio` apart, since currently they are aliases. He covered some of that in his [LSFMM session](<https://lwn.net/Articles/893512/>) on the previous day. 

Right now, you can simply cast a folio pointer to a page pointer and vice versa; "it's a bad code smell, but it works". The `page` structure has a pointer in disguise called "`memdesc`" that points to a `folio` structure. But there will need to be a way to get the page frame number (PFN) of the memory referred to by the folio once this 3-5 year "gargantuan project" of switching over to folios has finished. 

So there will need to be a way to go from a `folio` structure to the memory it is describing. Once that is working, there could be multiple folios allocated for the same PFN, but with a different mapping and index. That could lead to a solution to the problem of tracking the inodes associated with the cache entries; there could be multiple folios in different address spaces that all refer to the same memory. It is only a long-term solution, he said, because all of the filesystems will need to be changed to use folios before it can happen. 

Bacik said that he liked the idea of having a folio per inode that was sharing extents. But he wondered if that solution would be unpopular with the MM developers because pages with lots of references will seem like unattractive targets for reclaim, but these pages are simply in a cache that can be reclaimed. Kent Overstreet said that there needs to be a way to get a clear understanding of what a given chunk of memory is. The `page` structure cannot point to multiple folios, but it could point to a special kind of shared folio type that lists all of the folios that refer to the page. That shared folio could be put onto the least-recently-used (LRU) lists. Wilcox said that made sense to him. 

It is in some ways like kernel same-page merging (KSM), Johannes Weiner said; a `page` structure is what appears on the LRU and the MM code consults the container of that page to reclaim all of the mappings to it. But if every filesystem has to deal with walking the list of folios when reclaim needs to be done, that will make it harder to implement. Wilcox said that he originally thought it made sense for the filesystems to keep track of that information, but he was coming around to the idea that it should be done in the MM subsystem. Bacik said he would be happy as long as Wilcox did all the work to make it happen, which elicited laughter; Wilcox seemed to agree, however. 

Weiner said that KSM could use that facility as well, which Wilcox said "would be fantastic". KSM would just become another example of "something that is shared between multiple files". There was general agreement in the room on that approach. "OK, we solved it, thanks everyone for coming", Wilcox said with a laugh; there is, of course, a lot of work to be done to get there.
