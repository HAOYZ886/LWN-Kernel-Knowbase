---
title: Ext3 for large filesystems
url: https://lwn.net/Articles/187321/
date: "June 12, 2006"
category: "Filesystems-ext3"
author: 
---

Linux supports a wide variety of filesystems. While it is true that the Linux VFS layer treats all filesystems equally, the ext3 filesystem is certainly the first among equals. Ext3 is the default choice for a large majority of distributions; it can thus be found on vast numbers of installed Linux systems. If any filesystem were to be named _the_ Linux filesystem, it would be ext3. 

Ext3 is based on decades of experience with Unix filesystems. As a result, it is relatively straightforward to understand and highly reliable in its operation. It is, however, also showing its age in a number of ways. One of those is the maximum size of the underlying device it can handle. This limit is a mere 8 TB. That is enough to hold most of our mail spools \- even before spam filtering - but it is a limit which is already affecting some users. With the size of contemporary disks, the creation of an 8 TB array is not an entirely outlandish thing to do now, and it will only become easier over time. 

There are a couple of reasons for this limit. One of them is the use of 32-bit block numbers within the filesystem - and signed 32-bit numbers at that. The ext3 code can only track 2 gigablocks, which, using a 4K block size, sets the limit at 8 TB. Switching to an unsigned type can double that limit, but that only pushes back the problem by about one year. Clearly, larger block numbers are required. 

The other problem has to do with how ext3 tracks the blocks associated with any given file. The ext3 `inode` structure contains an array of fifteen 32-bit pointers; the first twelve of those pointers contain the indexes of the first twelve blocks of the file. Thus, with a filesystem using 4K blocks, the first twelve pointers can describe a file of up to 48KB in length. If the file exceeds that length, an "indirect block" is created. This block is a big array of block pointers, holding the indexes for the next 1024 blocks; the 13th pointer in the `inode` structure tracks the location of this indirect block. Should that space not suffice, the 14th pointer is used for a double-indirect block - a block holding pointers to indirect blocks. Finally, the 15th pointer will be used for a triple-indirect block if need be. 

This arrangement is not too different from how Unix systems structured filesystems two decades or more ago. It imposes a per-file maximum size of about 4 TB - big, but perhaps limiting for today's hot applications (such as comprehensive, nationwide telephone call archival). It works well for small files but, as files get larger, this organization becomes increasingly inefficient. Keeping a pointer to every single block is expensive, both in terms of space usage and the time it can take to locate a specific file block. Since larger filesystems will tend to hold larger files, this overhead becomes increasingly limiting over time. 

A solution to these problems can be found in the [extents and 48-bit support patch set](<http://lwn.net/Articles/186933/>). These patches have been posted by Mingming Cao; many other developers - especially Alex Tomas - have worked on them as well. They change the way files are stored to make things more efficient, and to allow the filesystem to index the blocks on larger devices. 

The core of the patch is the support for extents. An extent is simply a set of blocks which are logically contiguous within the file and also on the underlying block device. Most contemporary filesystems put considerable effort into allocating contiguous blocks for files as a way of making I/O operations faster, so blocks which are logically contiguous within the file often are also contiguous on-disk. As a result, storing the file structure as extents should result in significant compression of the file's metadata, since a single extent can replace a large number of block pointers. The reduction in metadata should enable faster access as well. 

An ext3 filesystem mounted with the extents option enabled will handle files stored in the old way, using block pointers, as always. New files will be created using extents, however. In these files, the fifteen-pointer array described above is overlaid with a new data structure. There is a short header, followed by a few occurrences of this structure: 

```
struct ext3_extent {
    	__le32	ee_block;	/* first logical block extent covers */
    	__le16	ee_len;		/* number of blocks covered by extent */
    	__le16	ee_start_hi;	/* high 16 bits of physical block */
    	__le32	ee_start;	/* low 32 bits of physical block */
        };
```

Here, `ee_block` is the index (within the file, not on disk) of the first block covered by this extent. The number of blocks in the extent is stored in `ee_len`, and the pointer to the first of those blocks (on disk, now) lives in the combination of `ee_start` and `ee_start_hi`. By storing physical block numbers this way, ext3 can handle 48-bit block numbers - enough to index a 1024 PB device. That should be enough to last for a couple years or so. 

For files with few extents, all of the information can be stored within the on-disk inode itself. As the number of extents grows, however, the available space runs out. In that case, a form of indirect blocks is used; the in-inode extents array describes ranges of blocks holding extents arrays of their own. The tree of indirect extents blocks can grow to an essentially unlimited depth, allowing the filesystem to represent even very large, highly-fragmented files. 

Beyond extents, relatively little had to be done to prepare ext3 for 48-bit block addressing. The signed, 32-bit block numbers are gone, having been converted to the larger `sector_t` type. Some reserved space in the ext3 superblock has been grabbed to store the high 16 bits of some global block counts. Much of the tracking of free blocks within the filesystem is done using block numbers relative to the beginning of the block group, so that code did not need to change much at all. A few tweaks to the journaling code were required for it to be able to handle the larger block numbers. 

The end result is an enhancement to the ext3 filesystem which enables it to work with much larger devices. Existing filesystems can use the new features immediately with no dump-and-restore cycle. It would appear to be (nearly) universally agreed that these changes turn ext3 into a better filesystem. Whether that better filesystem should still be called ext3 is controversial, but that is a subject for another article.
