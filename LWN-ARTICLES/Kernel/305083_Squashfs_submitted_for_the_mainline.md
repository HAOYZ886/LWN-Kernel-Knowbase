---
title: Squashfs submitted for the mainline
url: https://lwn.net/Articles/305083/
date: "October 29, 2008"
category: "Filesystems-Compressed"
author: "By Jake Edge October 29, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
October 29, 2008

The [Squashfs compressed filesystem](<http://squashfs.sourceforge.net/>) is used in everything from Live CDs to embedded devices. Many or most distributions ship it in such situations, but squashfs has been maintained outside of the mainline kernel for years. That appears to be changing as it was recently [submitted for inclusion in the mainline](<http://lwn.net/Articles/305058/>) by Phillip Lougher. The reaction has been generally favorable, with Andrew Morton [requesting that Lougher move it forward](<https://lwn.net/Articles/305128/>): ""Please prepare a tree for linux-next inclusion and unless serious problems are pointed out I'd suggest shooting for a 2.6.29 merge."" So it seems like a good time to take a look at some of the features and capabilities of Squashfs. 

The basic idea behind Squashfs is to generate a compressed image of a filesystem or directory hierarchy that can be mounted as a read-only filesystem. This can be done to archive a set of directories or to store them on a smaller capacity device than would normally be required. The latter is used by both Live CDs and embedded devices to squeeze more into less. 

It has been nearly four years since Squashfs was last submitted to linux-kernel. Since that time, it has been almost completely rewritten based on comments from that attempt. In addition, it has gone through two filesystem layout revisions in part to allow for 64-bit sizes for files and filesystems. Another major change is to make the filesystem little-endian, so that it can be read on any architecture, regardless of endian-ness. 

The `mksquashfs` utility is used to create the image, which can then be mounted either via loopback (from a file) or from a regular block device. One of the features added since the original attempt to mainline Squashfs--to address complaints made at that time--is the ability to export a Squashfs filesystem via NFS. 

Squashfs uses gzip compression on filesystem data and metadata, achieving sizes roughly one-third that of an ext3 filesystem with the same data. The [performance](<http://tree.celinuxforum.org/CelfPubWiki/SquashFsComparisons>) is quite good as well, even when compared with the simpler cramfs--a compressed read-only filesystem already available with the kernel. According to Lougher, these performance numbers were gathered a number of years ago, with older versions of the code; newer numbers should be even better. 

Previously, some kernel developers were resistant to adding another compressed filesystem to the kernel, so Lougher outlines a number of reasons that Squashfs is superior to cramfs. Certainly support for larger files and filesystems is compelling, but the fact that cramfs is orphaned and unmaintained will likely also play a role. In addition, Squashfs supports many more "normal" Linux filesystem features like real inode numbers, hard links, and exportability. 

Morton had a laundry list of overall suggestions for making Squashfs better in the email referenced above, but documentation is certainly one of the areas that is somewhat lacking. In particular, Squashfs maintains its own cache, which puzzles Morton: 

Why not just decompress these blocks into pagecache and let the VFS handle the caching?? 

The real bug here is that this rather obvious question wasn't answered anywhere in the patch submission (afaict). How to fix that? 

Methinks we need a squashfs.txt which covers these things. 

One of the reasons that Squashfs doesn't use the page cache is that it allows for multiple block sizes, from 4K up to 1M, with a default of 128K. Better compression ratios can be achieved with a larger block size, but that doesn't work well with the page cache as Jorn Engel [notes](<https://lwn.net/Articles/305133/>): ""One of the problems seems to be that your blocksize can exceed page size and there really isn't any infrastructure to deal with such cases yet."" 

Lougher has moved the code into a [git repository](<http://git.kernel.org/?p=linux/kernel/git/pkl/squashfs-2.6.git;a=summary>), presumably in preparation to get it into linux-next. He notes that the [CE Linux Forum](<http://celinuxforum.org/>) has been instrumental in providing funding over the last four months to allow him to work on getting Squashfs into the mainline. With the additional testing that will come from being included in linux-next, it seems quite possible we could see Squashfs in 2.6.29.
