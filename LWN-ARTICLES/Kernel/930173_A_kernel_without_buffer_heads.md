---
title: A kernel without buffer heads
url: https://lwn.net/Articles/930173/
date: "May 1, 2023"
category: "Block layer-Buffer heads"
author: "By Jonathan Corbet May 1, 2023"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 1, 2023

No data structures found in the Linux kernel — at least, in any version that escaped from Linus Torvalds's development machine — are older than the buffer head. Like many other legacies from the early days of Linux, buffer heads have been targeted for removal for years. They persist, though, despite the problems they present. Now, Christoph Hellwig has posted [a patch series](<https://lwn.net/ml/linux-kernel/20230424054926.26927-1-hch@lst.de/>) that enables the building of a kernel without buffer heads — but the cost of doing so at this point will be more than most want to pay. 

The first public release of the Linux kernel was version 0.01, and [`struct buffer_head`](<https://elixir.bootlin.com/linux/0.01/source/include/linux/fs.h#L56>) was a part of it: 

```
struct buffer_head {
    	char * b_data;			/* pointer to data block (1024 bytes) */
    	unsigned short b_dev;		/* device (0 = free) */
    	unsigned short b_blocknr;	/* block number */
    	unsigned char b_uptodate;
    	unsigned char b_dirt;		/* 0-clean,1-dirty */
    	unsigned char b_count;		/* users using this block */
    	unsigned char b_lock;		/* 0 - ok, 1 -locked */
    	struct task_struct * b_wait;
    	struct buffer_head * b_prev;
    	struct buffer_head * b_next;
    	struct buffer_head * b_prev_free;
    	struct buffer_head * b_next_free;
        };
```

While the best disk drives available decades ago were nominally "fast", accessing data on disk was still slower, by several orders of magnitude, than accessing data in main memory. So the importance of caching file data was well understood long before Linux was born. The approach that was generally in use at that time was to cache disk blocks, with filesystem code operating on data in that cache; Torvalds followed that model with Linux. Thus, from the beginning, the Linux kernel included a "buffer cache" that held copies of blocks found on the system's disks. 

The `buffer_head` structure was the key to managing the buffer cache. The combination of the `b_dev` and `b_blocknr` fields uniquely identified which block a given buffer cache entry referred to, while `b_data` pointed to the cached data itself. The other fields tracked whether the block needed to be written back to disk, how many users it had, and more. It was a core part of the kernel's block I/O subsystem — and of its memory management code as well. 

Over time, it became clear that file caching could be done better if it were implemented as a cache of _file_ data, rather than of disk blocks. During the 1.3 development cycle, Torvalds began implementing a new feature known as the "page cache", which would manage pages of data from files, rather than disk blocks. A number of advantages came from that change; many operations on file data could avoid calling into the filesystem code entirely if that data could be found in the cache, for example. Caching data at a higher level better matched how that data was used, and the ability to cache full pages (generally eight times larger than the 512-byte block size typically found at that time) improved efficiency. 

The only problem was that the buffer cache was deeply wired into both the block subsystem and the filesystem implementations, so this cache continued to exist, alongside the page cache, for several more years until the two were unified. Even then, the buffer cache was at the core of the API used for block I/O. This was not optimal: filesystems worked hard to store data contiguously on disk, and the page cache could keep that data together in memory with at least page granularity, but the buffer-head interface required every I/O operation to be broken down into 512-byte blocks — each with its own `buffer_head` structure. That was a lot of overhead, much of which just added work for storage drivers, which had to try to reassemble larger chunks for reasonable I/O performance. 

The 2.5 development series (the last of the odd-number development kernels under the older model) addressed this problem by reworking the block layer around [a new data structure called the "bio"](<https://lwn.net/Articles/26404/>) that could represent block I/O requests more efficiently. Over the years, the bio has evolved considerably as the need to support ever-higher I/O rates has grown, but it still remains the way that block I/O requests are assembled and managed. 

Meanwhile, though, [`struct buffer_head` can still be found](<https://elixir.bootlin.com/linux/v6.3/source/include/linux/buffer_head.h#L52>) in current kernels. And, more to the point, a number of filesystems still use it. The role that buffer heads once played in cache management has long since ended, but they still handle an important task in parts of the kernel: tracking the mapping between data cached in memory and the location on persistent storage where that data lives. The kernel has a rather more modern interface ([iomap](<https://elixir.bootlin.com/linux/v6.3/source/include/linux/iomap.h#L90>)) for this purpose, but not all subsystems are using it. 

One of the holdouts is ext4, which still makes heavy use of buffer heads. This filesystem, of course, is derived from ext2, which first entered the kernel with the 0.99.7 release in early 1993. Ext2 was based on block pointers; each file would have a list associated with it containing the numbers of the blocks on disk holding that file's data. Such a layout, where each block on disk is a separate entity (even if the filesystem tries to keep them together) fits the buffer head model reasonably well. So it is not surprising the buffer heads were embedded deeply within ext2, and are still there 30 years later in ext4, even though ext4 [gained support for extents](<https://lwn.net/Articles/187321/>) — a rather more efficient representation of large files — in 2006. 

Buffer heads, clearly, still work, but they still add overhead to file I/O. They also present an obstacle to changes that developers want to make to the memory-management and filesystem layers, including the ongoing [folio work](<https://lwn.net/Articles/893512/>). So the desire to get rid of buffer heads, which has been present for a long time, seems to be getting stronger. 

But, as Hellwig's patch series shows, ext4 is not the only place where buffer heads persist. That series, after a bit of refactoring, adds a new `BUFFER_HEAD` configuration option that controls the compilation of buffer-head support. Any code that needs buffer heads will select that option; if a kernel is built without any code needing buffer heads, then the resulting kernel will not have that support. Such a kernel will be lacking a few important features, though, including the ext4 filesystem, but also F2FS, FAT, GFS2, HFS, ISO9660 (CDROM), JFS, NTFS, NTFS3, and the device-mapper layer. On the other hand, it is possible to build a buffer-head-free kernel that supports Btrfs and XFS. 

It seems unlikely that there will be many kernels built without buffer-head support in the near future. This work does, however, make it easier to see where the remaining users are, which should help to focus work toward getting rid of buffer heads for real. That job is still likely to take some time — one does not perform major surgery on a heavily used filesystem in a hurry — and it may accelerate the removal of some old and unloved filesystems ([like JFS](<https://lwn.net/Articles/920259/>)). One of these years, though, it will become possible to drop this core kernel data structure that has been there since the beginning.
