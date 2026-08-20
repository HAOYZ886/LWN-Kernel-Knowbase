---
title: Supporting larger block sizes in filesystems
url: https://lwn.net/Articles/974219/
date: "May 22, 2024"
category: Filesystems
author: "By Jake Edge May 22, 2024 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 22, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

In a combined storage and filesystem session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Luis Chamberlain led a discussion on filesystem support for block sizes larger than the usual 4KB page size, which followed up on [discussion from last year](<https://lwn.net/Articles/933437/>). While the session was meant to look at the intersection of larger block sizes with atomic block writes that avoid torn (partial) writes (which was also [discussed last year](<https://lwn.net/Articles/933015/>)), it mostly focused on the filesystem side. Over time, the block sizes offered by storage devices have risen from the original 512 bytes; Chamberlain wanted to discuss filesystem support for block sizes larger than 4KB. 

[ ![\[Luis Chamberlain\]](https://static.lwn.net/images/2024/lsfmb-chamberlain-sm.png) ](<https://lwn.net/Articles/974573/>)

Chamberlain started by saying that he wanted to gauge the interest of filesystem developers in adding large-block support to their filesystems; in order to do so, a filesystem needs to be able to support large folios. The next obvious targets for this work are [bcachefs](<https://bcachefs.org/>) and [EROFS](<https://lwn.net/Articles/934047/>). Matthew Wilcox said that the large-folio support for EROFS is mostly done at this point, though there are a few places where it still uses `struct page`, for decompression in particular. For supporting large block sizes, EROFS is ready, he said, but the full folio-conversion job is not yet complete. 

Adding this support will require a lot of testing, Chamberlain said; beyond that, fstests has some baked-in assumptions about block size that need to be fixed. Some of those problems were found when testing with page sizes larger than 4KB, so they have been fixed at this point, but others may be lurking. He warned that filesystems with their own test suites may also have those kinds of assumptions. 

Damien Le Moal said that [zonefs](<https://docs.kernel.org/filesystems/zonefs.html>) developers are also interested in adding support for large block sizes. There are no fstests for zonefs, which Chamberlain suggested would be a useful addition to the suite. But Wilcox noted that zonefs uses [iomap](<https://kernelnewbies.org/KernelProjects/iomap>), so there is probably little work that needs to be done. It is mostly a matter of making a few calls to tell iomap that zonefs wants to use large folios. Le Moal said that large-folio support was being actively worked on for zonefs and was close to being ready. 

Iomap only handles the data path, Jan Kara pointed out, not the metadata path, which will still need more work for large blocks. Chamberlain agreed that was an outstanding problem. He wondered if filesystem developers even wanted to tackle it, because if they did not, there was not much point in going down the large-block path. 

Josef Bacik said that Btrfs is backward from other filesystems; by default it uses 16KB blocks for metadata, so it is the data path that needs converting. The plan is for Btrfs to switch to using iomap, then to turn on support for large folios "and pray". At that point, though, Btrfs should be able to use larger blocks. The iomap conversion is in progress, with direct I/O working now; buffered I/O is next. The support for the metadata follows what XFS has done, Bacik said. 

That led Dave Chinner to suggest that the XFS buffer-cache code be turned into a library that other filesystems can also use. Iomap came out of a similar process. Chamberlain wondered what other filesystems could benefit. Chinner said that any filesystem that uses the (deprecated) [buffer heads](<https://docs.kernel.org/next/filesystems/buffer.html>) API; the XFS buffer cache can support up to 64KB block sizes and already uses large folios. It could be pulled out of XFS, as it is already fairly generic; it is a wrapper around the page and slab allocators that provides compound buffers, which are made up of multiple discontiguous blocks but treated as a single contiguous buffer. Filesystems like ext4 that use buffer heads could be adapted to use this buffer cache in a fairly straightforward way. 

Neal Gompa thanked the developers working on making it easier to support larger block sizes in more filesystems, in part because he works on different distributions. He has encountered lots of problems trying to use filesystems that were created on distributions that use a different page size or block size on other distributions that made different choices. But, the terms he used for larger groups of pages, a superblock or superpage, were not popular; James Bottomley said that "superblock" was confusing because of its long-established use for filesystems, while Wilcox pointed out that a superpage should simply be called a folio. 

Ritesh Harjani asked about the benefits of supporting larger block sizes in filesystems, apart from the portability considerations. Chamberlain said that the hardware vendors are driving the move to larger blocks, but that he wanted to stick with the software side. He thinks larger blocks will help reduce file fragmentation, but deferred to the filesystem developers in the room. 

Darrick Wong said that he would actually like to get rid of discontiguous buffers for XFS because they are difficult to work with and to test, since they "cause all sorts of weird bugs to show up" in fstests. His advice to the other filesystem developers is: "try not to do that". It is not truly desirable to have metadata scattered in memory that way anyway. 

He has some patches for [fs-verity](<https://docs.kernel.org/filesystems/fsverity.html>) support for XFS "stuck in the three-mile freight train of everything that's in my development tree that's blocking traffic all over the city". As part of that, he found a need for a buffer cache, so he reused some of the XFS code for it. That work could be used as the basis of a new library for filesystem metadata handling as Chinner had suggested. He is trying to figure out how to integrate ("staple") that work onto the [jbd2 journal layer](<https://www.kernel.org/doc/html/latest/filesystems/ext4/journal.html>); doing that would mean that ext4 could use it, but that also requires porting [OCFS2](<https://docs.kernel.org/filesystems/ocfs2.html>) to use the new buffer cache. Since he believes no one actually uses OCFS2, perhaps the filesystem could just be deprecated instead. 

Before even setting up iomap, though, there needs to be a mechanism to read from the disk, Hannes Reinecke said. Chamberlain suggested using iomap to read that data, but Reinecke insisted that it cannot read the data for the superblock, from which iomap can be configured. That requires buffer heads. But Chamberlain said that block-device operations can be used to retrieve the needed superblock data, thus buffer heads were not required. He agreed that more discussion on that was needed, however.
