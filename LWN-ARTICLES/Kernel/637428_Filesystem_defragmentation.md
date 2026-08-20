---
title: Filesystem defragmentation
url: https://lwn.net/Articles/637428/
date: "March 23, 2015"
category: Filesystems
author: "By Jake Edge March 23, 2015 LSFMM 2015"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
March 23, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

Dmitry Monakhov prefaced his 2015 LSFMM Summit session on filesystem defragmentation with a statement that the "problem is almost already solved". His session turned into a largely informational description of the status of a defragmentation tool that he has been working on. 

Over time, filesystems change and cannot avoid fragmentation issues, he said. For example, extracting a Linux source tree results in many small files that filesystem tries to allocate close to each other. Building in the tree results in lots of temporary files that get removed, so the filesystem gets fragmented. 

Beyond appearing in regular filesystems, these fragmentation problems show up in thin provisioning systems, as well as for shingled magnetic recording (SMR) devices, he said. In addition, to make boot times shorter, it would be best to lay out all the needed files sequentially on the disk, which may require defragmentation. 

The fragmentation problem is already solved for large files. Btrfs, XFS, and ext4 all have tools for doing defragmentation on files. But there is no solution for directory fragmentation. The filesystems try to put files that are in the same directory close to each other on the disk, but as files get deleted or moved, fragmentation of the directory occurs. 

To perform defragmentation, it is often necessary to copy file data from one place to another. Monakhov suggested that a checksum could be calculated on the data when doing that copy, which could then be stored in a "trusted" extended attribute (xattr). He noted that overlayfs uses the "trusted.overlay" xattr, which can only be modified by processes with `CAP_SYS_ADMIN`, so a "trusted.sha1" (or or other hash) could be calculated and stored when copying data for defragmentation. 

Executable files could then have their contents checked and compared to the hash value before being executed. He proposed adding that capability to his tool, but it seemed to be something of an aside. It is not clear how it relates to the [integrity measurement architecture (IMA)](<https://lwn.net/Articles/488906/>), for example. 

He has been working on a tool called `e4defrag2` (developed in a [branch of e2fsprogs](<https://github.com/dmonakhov/e2fsprogs/tree/e4defrag2>)) that will perform defragmentation. It is mostly independent of the filesystem type. It uses the same block scanning code to find fragmentation, but ext4 and XFS have a different `ioctl()` name for their defragmentation operations. 

The result is a "giant utility that works for everything", Monakhov said. The filesystem-dependent part is roughly 100 lines of code. This "universal defragmenter" will be released soon. 

Ted Ts'o asked what would be needed to eliminate the 100 lines. He asked if wiring up the XFS `ioctl()` name into ext4 would help. Monakhov said that the tool needs to get the block bitmap from the filesystem, which is also different between the filesystems. Ts'o and Dave Chinner indicated that they would attempt to provide the same interfaces. Chinner did caution that XFS cannot defragment a range in a file, only the whole file. That is different than ext4, Monakhov said. 

[I would like to thank the Linux Foundation for travel support to Boston for the summit.]
