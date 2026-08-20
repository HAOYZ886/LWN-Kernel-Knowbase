---
title: Filesystems and iomap
url: https://lwn.net/Articles/974958/
date: "May 28, 2024"
category: "Filesystems-iomap"
author: "By Jake Edge May 28, 2024 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 28, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The [iomap](<https://kernelnewbies.org/KernelProjects/iomap>) block-mapping abstraction is being used by more filesystems, in part because of its support for large folios. But there are some challenges in adopting iomap, which was the topic of a discussion led by Ritesh Harjani in a combined storage and filesystem session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). One of the main trouble spots is how to handle metadata, which is not an area that iomap has been aimed at. 

Iomap has become a VFS abstraction for mapping logical offset ranges to the physical extents for files, Harjani began. It provides an iterator model that is filesystem-centric, rather than page-cache-centric, with regard to mapping the byte offset of a file to its blocks on the storage device. It also abstracts common page-cache operations and supports mapping from folios (and large folios). 

[ ![\[Ritesh Harjani\]](https://static.lwn.net/images/2024/lsfmb-harjani-sm.png) ](<https://lwn.net/Articles/975448/>)

Managing the "dirty" flags for individual blocks was not possible with large folios, because there was only a single bit to track that state for the entire folio. Per-block tracking of that state has been added, so that only those blocks that are actually dirty need to be handled during writeback. That provides a significant savings by avoiding the write amplification that happened during writeback without the tracking, he said, which means that iomap scales much better than before. 

Harjani then talked about the upstream status of various pieces. Ext4 and Btrfs both switched to using iomap for direct I/O, in Linux 5.5 and 5.8, respectively. The 6.6 kernel added large-folio support and per-block dirty tracking to iomap, as well as iomap for ext2 direct I/O. In 6.9, the multi-block mapping optimization for iomap was added; it allows specifying a range of dirty blocks for writeback. 

There are various things in progress as well. Ext2 buffered-I/O iomap-conversion patches are in-process at this point, while the ext4 buffered-I/O conversion, for filesystems mounted with default options, is being worked on. There is work going on to optimize access to filesystems that have indirect block mappings. Getting iomap documentation into the kernel tree was [discussed at last year's summit](<https://lwn.net/Articles/935934/>), but has not yet happened; he has a [documentation patch](<https://lwn.net/ml/linux-fsdevel/17e84cbae600898269e9ad35046ce6dc929036ae.1714744795.git.ritesh.list@gmail.com/>) that is out for review, so that problem should be solved relatively soon. 

There are a number of things that are motivating filesystem developers to make the switch to iomap. Support for buffered atomic writes in iomap is in the works, as well as support for block sizes larger than the system's page size. Beyond that, Matthew Wilcox noted that if developers switch their filesystem, "I will stop bugging you about large-folio support"; XFS uses iomap and no longer deals with pages at all. "Folios, pages, you don't care anymore if you use iomap." 

There is a long list of filesystems that have at least some support for iomap, Harjani said, based on a search for "`FS_IOMAP`" in the `fs` tree. He did the same search for "`LEGACY_DIRECT_IO`" to show filesystems that have no support for iomap and wondered what the plan in the filesystem community was for those. Al Viro said that the `LEGACY_DIRECT_IO` search was not really the right way to look at the problem, because it artificially splits filesystems that are not that different from each other. There are several filesystems on the list that could directly benefit from the work that has been done on ext2, for example, but only if someone actually cares enough about them to do it. He may look into adapting the ext2 work for minixfs. 

Amir Goldstein wondered who was going to test any changes to the legacy filesystems, many of which do not have any tests—or even a way to create a filesystem (`mkfs`). Harjani said that he thought the person doing the conversion would work with the maintainer, but Goldstein said that some of the maintainers do not really have the time to work on things of that sort. It goes back to the problem of [unmaintained filesystems](<https://lwn.net/Articles/939097/>) in the kernel, which has been a recurring topic at the summit over the years. 

In his conversion of ext2, Harjani found that the directory-handling code uses the page cache directly. Iomap does not export an API that is similar to the byte-oriented API that ext2 currently uses. Perhaps iomap can export an API that can be used for that. 

There is no support for metadata I/O at all in iomap. One possible solution is to lift the buffer-cache code from XFS, as was discussed in the [large-block-size session](<https://lwn.net/Articles/974219/>) earlier in the day. Another solution would be to do some "surgery" on the [buffer-head API](<https://docs.kernel.org/next/filesystems/buffer.html>). That would require adding ways to read metadata blocks, track metadata buffers that are not attached to an inode, and to track buffers for journaling them before doing I/O to the filesystem. He wondered which approach made more sense. 

Iomap was never intended for metadata use, Dave Chinner said, as a bit of background. It was only ever used for the data path in XFS, which is where iomap came from. He is not sure that trying to use iomap for metadata is the right approach; metadata handling is typically filesystem-specific, such as for journaling. He thinks that looking at a replacement for buffer heads would be the right mechanism for metadata handling. Harjani thought that iomap had features that made it attractive to use for metadata; Chinner was somewhat skeptical but thought it probably could be done. 

For ext2, the metadata is the directory contents, Viro said; the indirect blocks are another kind of metadata for the filesystem, but Harjani said he was focused on the directory contents. Viro suggested treating a directory as much like a regular file as possible; it would be strange to use iomap for files, but not for directories, because they are close to the same thing. 

Wilcox said that unifying the page cache and buffer cache, and using the page cache for directories, was a mistake. He thinks there should be a separate buffer cache for the directory information; the page cache keeps a "lot of metadata about metadata" that is unneeded. For example, you do not need a `map_count` for a directory, because it cannot be mapped using [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>). 

What is really needed is a buffer cache that is not just an alias into the page cache, which is something that he thinks XFS developer Darrick Wong has already done. Viro said that historically using the same layer for file data and metadata, such as the page cache, has been the norm, possibly going all the way back to [Multics](<https://en.wikipedia.org/wiki/Multics>). Wilcox said that the page cache exists, in part, to ensure that writes and `mmap()` do not interfere with each other, which is not a problem for directories. 

In conclusion, Harjani said that he would pursue the iomap approach for metadata to see where that goes for ext2. He would also like to see the buffer-head interface get stripped down to what is essential and be renamed to fs_buf or something along those lines.
