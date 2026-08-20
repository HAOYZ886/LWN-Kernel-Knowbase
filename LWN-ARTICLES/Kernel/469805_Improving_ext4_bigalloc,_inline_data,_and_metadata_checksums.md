---
title: "Improving ext4: bigalloc, inline data, and metadata checksums"
url: https://lwn.net/Articles/469805/
date: "November 29, 2011"
category: "Filesystems-ext4"
author: "By Jonathan Corbet November 29, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 29, 2011

It may be tempting to see ext4 as last year's filesystem. It is solid and reliable, but it is based on an old design; all the action is to be found in next-generation filesystems like Btrfs. But it may be a while until Btrfs earns the necessary level of confidence in the wider user community; meanwhile, ext4's growing user base has not lost its appetite for improvement. A few recently-posted patch sets show that the addition of new features to ext4 has not stopped, even as that filesystem settles in for a long period of stable deployments. 

#### Bigalloc

In the early days of Linux, disk drives were still measured in megabytes and filesystems worked with blocks of 1KB to 4KB in size. As this article is being written, terabyte disk drives are not quite as cheap as they recently were, but the fact remains: disk drives have gotten a lot larger, as have the files stored on them. But the ext4 filesystem still deals in 4KB blocks of data. As a result, there are a lot of blocks to keep track of, the associated allocation bitmaps have grown, and the overhead of managing all those blocks is significant. 

Raising the filesystem block size in the kernel is a dauntingly difficult task involving major changes to memory management, the page cache, and more. It is not something anybody expects to see happen anytime soon. But there is nothing preventing filesystem implementations from using larger blocks on disk. As of the 3.2 kernel, ext4 will be capable of doing exactly that. The "bigalloc" patch set adds the concept of "block clusters" to the filesystem; rather than allocate single blocks, a filesystem using clusters will allocate them in larger groups. Mapping between these larger blocks and the 4KB blocks seen by the core kernel is handled entirely within the filesystem. 

The cluster size to use is set by the system administrator at filesystem creation time (using a development version of e2fsprogs), but it must be a power of two. A 64KB cluster size may make sense in a lot of situations; for a filesystem that holds only very large files, a 1MB cluster size might be the right choice. Needless to say, selecting a large cluster size for a filesystem dominated by small files may lead to a substantial amount of wasted space. 

Clustering reduces the space overhead of the block bitmaps and other management data structures. But, as Ted Ts'o [documented](<https://lwn.net/Articles/469821/>) back in July, it can also increase performance in situations where large files are in use. Block allocation times drop significantly, but file I/O performance also improves in general as the result of reduced on-disk fragmentation. Expect this feature to attract a lot of interest once the 3.2 kernel (and e2fsprogs 1.42) make their way to users. 

#### Inline data

An inode is a data structure describing a single file within a filesystem. For most filesystems, there are actually two types of inode: the filesystem-independent in-kernel variety (represented by `struct inode`), and the filesystem-specific on-disk version. As a general rule, the kernel cannot manipulate a file in any way until it has a copy of the inode, so inodes, naturally, are the focal point for a lot of block I/O. 

In the ext4 filesystem, the size of on-disk inodes can be set when a filesystem is created. The default size is 256 bytes, but the on-disk structure (`struct ext4_inode`) only requires about half of that space. The remaining space after the `ext4_inode` structure is normally used to hold extended attributes. Thus, for example, SELinux labels can be found there. On systems where extended attributes are not heavily used, the space between on-disk inode structures may simply go to waste. 

Meanwhile, space for file data is allocated in units of blocks, separately from the inode. If a file is very small (and, even on current systems, there are a lot of small files), much of the block used to hold that file will be wasted. If the filesystem is using clustering, the amount of lost space will grow even further, to the point that users may start to complain. 

Tao Ma's [ext4 inline data patches](<https://lwn.net/Articles/468678/>) may change that situation. The idea is quite simple: very small files can be stored directly in the space between inodes without the need to allocate a separate data block at all. On filesystems with 256-byte on-disk inodes, the entire remaining space will be given over to the storage of small files. If the filesystem is built with larger on-disk inodes, only half of the leftover space will be used in this way, leaving space for late-arriving extended attributes that would otherwise be forced out of the inode. 

Tao says that, with this patch set applied, the space required to store a kernel tree drops by about 1%, and `/usr` gets about 3% smaller. The savings on filesystems where clustering is enabled should be somewhat larger, but those have not yet been quantified. There are a number of details to be worked out yet - including e2fsck support and the potential cost of forcing extended attributes to be stored outside of the inode - so this feature is unlikely to be ready for inclusion before 3.4 at the earliest. 

#### Metadata checksumming

Storage devices are not always as reliable as we would like them to be; stories of data corrupted by the hardware are not uncommon. For this reason, people who care about their data make use of technologies like RAID and/or filesystems like Btrfs which can maintain checksums of data and metadata and ensure that nothing has been mangled by the drive. The ext4 filesystem, though, lacks this capability. 

Darrick Wong's [checksumming patch set](<https://lwn.net/Articles/469717/>) does not address the entire problem. Indeed, it risks reinforcing the old jest that filesystem developers don't really care about the data they store as long as the filesystem metadata is correct. This patch set seeks to achieve that latter goal by attaching checksums to the various data structures found on an ext4 filesystem - superblocks, bitmaps, inodes, directory indexes, extent trees, etc. - and verifying that the checksums match the data read from the filesystem later on. A checksum failure can cause the filesystem to fail to mount or, if it happens on a mounted filesystem, remount it read-only and issue pleas for help to the system log. 

Darrick makes no mention of any plans to add checksums for data as well. In a number of ways, that would be a bigger set of changes; checksums are relatively easy to add to existing metadata structures, but an entirely new data structure would have to be added to the filesystem to hold data block checksums. The performance impact of full-data checksumming would also be higher. So, while somebody might attack that problem in the future, it does not appear to be on anybody's list at the moment. 

The changes to the filesystem are significant, even for metadata-only checksums, but the bulk of the work actually went into e2fsprogs. In particular, e2fsck gains the ability to check all of those checksums and, in some cases, fix things when the checksum indicates that there is a problem. Checksumming can be enabled with mke2fs and toggled with tune2fs. All told, it is a lot of work, but it should help to improve confidence in the filesystem's structure. According to Darrick, the overhead of the checksum calculation and verification is not measurable in most situations. This feature has not drawn a lot of comments this time around, and may be close to ready for inclusion, but nobody has yet said when that might happen.
