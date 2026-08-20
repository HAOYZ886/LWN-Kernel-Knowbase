---
title: "get_user_pages(), pinned pages, and DAX"
url: https://lwn.net/Articles/787636/
date: "May 7, 2019"
category: "Memory management-get user pages"
author: "By Jonathan Corbet May 7, 2019 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 7, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The problems associated with the kernel's internal `get_user_pages()` function have been a topic of discussion at the Linux Storage, Filesystem, and Memory-Management Summit for a few years. At the 2019 event, Jan Kara began a plenary session by saying that it would be "like last year's session". It turned out rather differently, though, perhaps due to the plenary setting; this discussion (along with the related session that followed) turned out to be one of the most heated at the entire conference. 

#### Pinned pages

Kara said that he didn't really want to waste a lot of time explaining the problem yet again, so he went over the background in a hurry; readers who are not familiar with the problem can learn more in [this article](<https://lwn.net/Articles/784574/>). In the end, it comes down to filesystems that are unprepared for pages of data to be modified behind their back, leading to problems like data corruption and kernel crashes. 

He has developed a list of frequently asked questions over the years of working on this problem. Could `get_user_pages()` simply be disabled for file-backed pages? No, because real applications use it. They map shared buffers for direct I/O, for example; these applications were written before the addition of the [`splice()`](<http://man7.org/linux/man-pages/man2/splice.2.html>) system call, but they are still being used. Could the `PageLocked` flag be used to lock the pages entirely? That would lead to pages being locked for long periods of time, and could deadlock the system in a number of scenarios. How about using MMU notifiers to get callers to drop their [![\[Jan Kara\]](https://static.lwn.net/images/conf/2019/lsfmm/JanKara-sm.jpg)](<https://lwn.net/Articles/787642/>) references to pages on demand? "Good luck converting over 300 call sites" to the new scheme, Kara said; that would also increase the overhead for short-lived mappings that are used for tasks like direct I/O. 

So something else must be done. The plan, Kara said, is to create a new type of page reference for `get_user_pages()` called a "pin". It would be obtained by calling `get_pin()`, which would increase the target page's reference count by a large bias value. Then, any page with a reference count greater than the bias value will be known to have pin references to it. There are some details to deal with, including the possible overflow of the reference count, though it shouldn't be a problem for real use cases. A large number of references could occasionally cause false positives, but that would not create problems either. 

Kirill Shutemov suggested just making the `PagePinned` flag reliable, but Kara responded that doing so would require adding another reference count for each page. Space is not readily available for such a count. He had actually looked at schemes involving taking pinned pages out of the least-recently-used (LRU) lists, at which point the list pointers could be used, but it was not an easy task and conflicted with what the heterogeneous memory management code is doing. 

The next step is to convert `get_user_pages()` users to release their pages with `put_user_page()`, which has already been merged into the mainline. There are a lot of call sites, so it will take a while to get this job done. 

Christoph Hellwig jumped in to say that there are other problems with `get_user_pages()`, and that this might be a good time to do a general spring cleaning. Many of the users, he said, are "bogus"; many of the callers never actually look at the pages they have mapped. For the cases where a scatter/gather list is needed for DMA I/O, a helper could be provided. But Jérôme Glisse said that it would not, in the end, be possible to remove that many callers. 

Once the system can tell which pages are pinned, it's just a matter of deciding what to do with that information. Kara talked mostly about the writeback code; in some cases, it will simply skip pages that are pinned. But there are cases where those pages must be written out — "somebody has called `fsync()`, and they expect something to be saved". In this case, pinned pages will be written, but they will not be marked clean at the end of the operation; they will still be write-protected in the page tables while writeback is underway, though. In situations where stable pages are required (when a checksum of the data must be written with the data, for example), bounce buffers will be used. There are over 40 places to fix up, not all of which are trivial. 

#### DAX and long-term pins

At this point, Ira Weiny took over the leadership of the discussion to talk about the problems related to long-lasting page pins and persistent memory in particular. The DAX interface allows user space to operate directly on the memory in which files are stored (when the filesystem is on a persistent-memory device), which brings some interesting challenges. Writeback is not normally a problem with DAX, he said, but other operations, such as truncation and hole-punching are. When pages are removed from a file, they are normally taken out of the page cache as well, but with DAX those pages were never in the page cache to begin with. Instead, users are dealing directly with the storage media. 

As a result, a truncate operation may have removed pages from a file while some of the pages in that file are pinned in memory. In this case, the filesystem has to leave the pages in place on the device. When pages are [![\[Ira Weiny\]](https://static.lwn.net/images/conf/2019/lsfmm/IraWeiny-sm.jpg)](<https://lwn.net/Articles/787643/>) pinned for a long time (as RDMA will do, for example), those pages can be left there indefinitely. 

Weiny said that what is needed to solve this problem is some sort of a lease mechanism so that pinned pages can be unpinned on demand. He has a prototype implementation working now; it implements leases with `get_user_pages()`, with individual call sites being able to indicate whether they support this mechanism. If a user has a file mapped and pages are removed from it, that user will get a `SIGBUS` signal to indicate that this has happened. Weiny asked whether the group thought this approach was reasonable. 

One attendee noted that NFS seems to handle this case now; it can lose a file delegation on a truncate event. Perhaps the kernel should use the NFS (or the SMB direct) mechanism? There are challenges to implementing that, Weiny said, and in any case it's not the approach that people have been looking at. DAX is fundamentally different, he said, in that it uses a filesystem to control memory semantics. 

Boaz Harrosh said that the approach was wrong, and that the truncate or hole-punch operation should simply fail when pinned pages are present. Others agreed that it wasn't right to just randomly kill processes because somebody truncated a file somewhere. Whether that is truly "random" was a matter of debate, but that was a secondary issue. 

The rest of the session was an increasingly loud argument between those who favored sending `SIGBUS` and those who thought that the file operation should fail. At one point Hellwig suggested that people who were yelling really didn't need to use the microphone as well. Your editor will confess to having simply stopped taking notes partway through; it was one of those disagreements where opinions are strong and nobody is prepared to change their mind. About the only point of agreement was that the current behavior in this situation, wherein a call to `truncate()` will simply hang, is not good. 

Toward the end, Ted Ts'o said that it would probably prove necessary to implement both options sooner or later. The session ended, rather later than scheduled, with no agreement as to what the right solution is. This topic seems likely to light up the mailing lists in the future.
