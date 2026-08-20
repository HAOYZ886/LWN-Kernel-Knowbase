---
title: A NILFS2 score card
url: https://lwn.net/Articles/522507/
date: "November 7, 2012"
category: "Filesystems-Flash"
author: "November 7, 2012 This article was contributed by Neil Brown"
---

November 7, 2012

This article was contributed by Neil Brown

A recurring [theme](<https://lwn.net/Articles/518820/>) in the [comments](<http://hardware.slashdot.org/comments.pl?sid=3166405&cid=41565197>) on various [articles](<http://www.reddit.com/r/programming/comments/11ktzg/a_deep_and_detailed_look_at_samsungs_f2fs/c6nnv4k>) announcing the new [`f2fs`](<https://lwn.net/Articles/518988/>) "flash-friendly" filesystem was that surely some other filesystem might already meet the need, or could be adjusted to meet the need, rather than creating yet another new filesystem. This is certainly an interesting question, but not one that is easy to answer. The cost/benefit calculation for creating a new filesystem versus enhancing an existing one involves weighing many factors, including motivational, social, political, and even, to some extent, technical. It is always more fun building something from scratch; trying to enter and then influence an existing community is at best unpredictable. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Of the various factors, the only ones for which there is substantial visibility to the outside observer are the technical factors, so while they may not be the major factors, they are the ones that this article will explore. In particular, we will examine "`NILFS2`", a filesystem which has been part of Linux for several years and is -- superficially at least -- one the best contenders as a suitable filesystem for modest-sized flash storage. 

`NILFS2` was not written primarily to be a flash-based filesystem, so comparing it head-to-head on that basis with `f2fs` (which was) might not be entirely fair. Instead we will examine it on its own merits, comparing it with `f2fs` occasionally to provide useful context, and ask "could this have made a suitable base for a flash-focused filesystem?" 

#### `NILFS2`: what is it?

`NILFS2` is the second iteration of the "[New Implementation of a Log-structured File System](<http://www.nilfs.org/en/about_nilfs.html>)". It is described as a "Continuous Snapshotting" filesystem, a feature which will be explored in more detail shortly. 

`NILFS2` appears to still be under development, with lots of core functionality present, but a number of important features still missing, such as extended attributes, quotas, and an `fsck` tool. As such, it is in a similar state to `f2fs`: well worth experimenting with, but not really ready for production usage yet. 

In contrast with `f2fs`, NILFS2 uses 64 bits for all block addressing, 96 bits for time stamps (nanoseconds forever!), but only 16 bits for link counts (would _you_ ever have more than 65535 links to a file, or sub-directories in one directory?). `F2fs`, in its initial release, uses 32 bits for each of these values. 

While `f2fs` is a hybrid LFS ([Log-structured Filesystem](<http://en.wikipedia.org/wiki/Log-structured_file_system>)), using update-in-place in a number of cases, `NILFS2` is a pure LFS. With the exception of the superblock (stored twice, once at either end of the device), everything is written in one continuous log. Data blocks are added to the log, then indexing information, then inodes, then indexes for the inodes, and so on. Occasionally a "super root" inode is written, from which all other blocks in the filesystem can be found. The address of the latest "super root" is stored in the superblock, along with static values for various parameters of the filesystem and a couple of other volatile values such as the number of free blocks. 

Whenever a collection of blocks is written to the log, it is preceded by a segment summary which identifies all the blocks in the segment (similar to the Segment Summaries of `f2fs` which are stored in a separate area). Consecutive segment summaries are linked together so that, in the event of a crash, all the segment summaries since the most recent super root can be read and the state of the filesystem can be reconstructed. 

The segment size can be chosen to be any number of blocks, which themselves must have a size that is a power of two up to the page size of the host system. The default block size is 4KB and the default device segment size is 8MB. Segments can easily be made to line up with erase blocks in a flash device, providing their size is known. While `NILFS2` tries to write whole segments at a time, it is not always possible, so a number of consecutive partial segments might be written, each with their own segment summary block. 

Being a pure LFS, `NILFS2` will never write into the middle of an active segment -- as `f2fs` does when space is tight. It insists on "cleaning" partially used segments (copying live data to a new segment) to make more space available, and does not even keep track of which particular blocks in a segment might be free. If there are no clean segments beyond those reserved for cleaning, the filesystem is considered to be full. 

> ![\[Cleaning\]](https://static.lwn.net/images/2012/nilfs-clean.png)

#### Everything is a file!

The statement "Everything is a file" is part of the Unix tradition and, like many such statements, it sounds good without tying you down to meaning very much. Each of "file", "everything" and even "is a" is open to some interpretation. If we understand "file" to be a collection of data and index blocks that provide some linearly addressed storage, "everything" to mean most data and metadata -- excepting only the superblock and the segment summaries -- and "is a" to mean "is stored inside a", then `NILFS2` honors this Unix tradition. 

For example, in a more traditional filesystem such as ext3, inodes (of which there is one per file) are stored at fixed locations in the device -- usually a number of locations distributed across the address space, but fixed nonetheless. In `f2fs`, a hybrid approach is used where the addresses of the inodes are stored in fixed locations (in the Node Address Table -- NAT), while the inodes themselves appear in the log, wherever is convenient. For `NILFS2`, the inode table is simply another file, with its own inode which describes the locations of the blocks. 

This file (referred to as the `ifile`) also contains a bitmap allowing unused inodes to be found quickly, and a "block group descriptor" which allows non-empty bitmaps to be found quickly. With the default block size, every 225 inodes has 1024 blocks for bitmaps, and one descriptor block, which lists how many bits are set in each of those bitmaps. If you want more inodes than that, a second descriptor block will be automatically allocated. 

> ![\[Ifile\]](https://static.lwn.net/images/2012/nilfs-ifile.png)

The inodes themselves are a modest 128 bytes in size, and here I must confess to an oversimplification in the article on `f2fs`. The statement that "Copy-on-write is rather awkward for objects that are smaller than the block size" holds a grain of truth, but isn't really true as it stands. The reality is more subtle. 

The advantages of a small inode size are primarily in efficiency. Less space can be wasted, and fewer I/O requests are needed to load the same number of inodes. For a traditional filesystem with pre-allocated inode regions, the space wasted can be a significant issue. However, that does not really apply to an LFS which allocates the space on demand. The speed issue is slightly harder to reason about. Certainly if all the inodes for files in one directory live in one block, then the common task of running "`ls -l`" will be expedited. However if more information, such as extended attributes or file data for small files, is stored in a big inode, accessing that will require only one block to be read, not two. 

The advantages of a block-sized inode -- apart from the extra space, which is of uncertain value -- is that inodes can be updated independently. OCFS2 (a cluster-based filesystem) uses this to simplify the locking overhead -- a cluster node does not need to gain exclusive access to inodes in the same block as the one that it is interested in when it performs an update, because there aren't any. In an LFS, the main issue is reducing cleaning overhead. As we noted in the f2fs article, grouping data with similar life expectancy tends to reduce the expected cost of cleaning, so storing an inode together with the data that it refers to is a good idea. If there are several inodes in the one block, then the life expectancy will be the minimum for all the inodes, and so probably quite different from nearby data. This could impose some (hard to measure) extra cleaning cost. 

On the whole, it would seem best for an LFS if the one-inode-per-block model were used, as there is minimal cost of wasted space and real opportunities for benefits. If ways are found to make maximal use of that extra space, possibly following some [ideas that Arnd Bergmann recently suggested](<https://lkml.org/lkml/2012/10/17/228>), then block-sized inodes would be even more convincing. 

Small inodes might be seen as a reason not to choose `NILFS2`, though not a very strong reason. Adjusting `NILFS2` to allow full-block inodes would not be a large technical problem, though it is unknown what sort of social problem it might be. 

As with most filesystems, `NILFS2` also stores each directory in a "file" so there are no surprises there. The surprise is that the format used is extremely simple. `NILFS2` directories are nearly identical to ext2 directories, the only important difference being that they store a 64-bit inode number rather than just 32 bits. This means that any lookup requires a linear search through the directory. For directories up to about 400 entries (assuming fairly short names on average), this is no different from `f2fs`. For very large directories, the search time increases linearly for `NILFS2`, while it is logarithmic for `f2fs`. While `f2fs` is not particularly efficient at this task, `NILFS2` clearly hasn't made any effort at effective support for large directories. There appears to be an intention to implement some sort of B-tree based directory structure in the future, but this has not yet happened. 

#### Use the right tree for the job

If everything is a file, then it is clearly important to know what a file is. It starts at the inode which contains space for seven 64-bit addresses. When the file is small (seven blocks or less) these contain pointers to all of the allocated blocks. When the file is larger, this space changes to become the root of a B-tree, with three keys (file addresses), three pointers to other B-tree nodes, and a small header. 

The interesting thing about this B-tree is that the leaves do not contain extents describing contiguous ranges of blocks; instead they describe each block individually. This is interesting because it does not fit the typical use-case for a [B-tree](<http://en.wikipedia.org/wiki/B-tree>). 

The particular value of a B-tree is that it remains balanced independently of the ordering or spacing of keys being inserted or removed. There is a cost that blocks may occasionally need to be split or merged, but this is more than compensated for by the ability to add an unpredictable sequence of keys. When extents are being stored in a tree, it is not possible to predict how long each extent will be, or when an extent might get broken, so the sequence of keys added will be unpredictable and a B-tree is ideal. 

When the keys being added to the index are the offsets of consecutive blocks, then the sequence is entirely predictable and a different tree is likely to be preferred. A radix tree (where the path through the tree is a simple function of the key value) is much more compact than a B-tree (as there is no need to store keys) and much simpler to code. This is the sort of tree chosen for `f2fs`, the tree used for ext3, and generally the preferred choice when block extents are not being used to condense the index. 

The only case where a B-tree of individual blocks might be more efficient than a radix tree is where the file is very sparse, having just a few allocated blocks spread throughout a large area that is unallocated. Sparse files are simply not common enough among regular files to justify optimizing for them. Nor are many of the special files that `NILFS2` uses likely to be sparse. The one exception is the checkpoint file (described later), and optimizing the indexing strategy for that one file is unlikely to have been a motivation. 

So we might ask "why?". Why does `NILFS2` use a B-tree, or why does it not use extents in its addressing? An early [design document [PDF]](<http://www.nilfs.org/papers/overview-v1.pdf>) suggests that B-trees were chosen due to their flexibility, and while it isn't yet clear that the flexibility is worth the cost, future developments might show otherwise. The lack of extent addressing can be explained with a much more concrete answer once we understand one more detail about file indexing. 

#### Another layer of indirection

The headline feature for `NILFS2` is "continuous snapshotting". This means that it takes a snapshot of the state of the filesystem "every few seconds". These are initially short-term snapshots (also called "checkpoints"), and can be converted to long-term snapshots, or purged, by a user-space process following a locally configurable policy. This means there are very likely to be lots of active snapshots at any one time. 

As has been mentioned, the primary cost of an LFS is cleaning -- the gathering of live data from nearly-empty segments to other segments so that more free space can be made available. When there is only one active filesystem, each block moved only requires one index to be updated. However, when there are tens or hundreds of snapshots, each block can be active in a fairly arbitrary sub-sequence of these, so relocating a block could turn into a lot of work in updating indices. 

Following the [maxim usually attributed to David Wheeler](<http://en.wikipedia.org/wiki/Indirection>), this is solved by adding a level of indirection. One of the special files that `NILFS2` uses is known as the "DAT" or "Disk Address Translation" file. It is primarily an array of 64-bit disk addresses, though the file also contains allocation bitmaps like the `ifile`, and each entry is actually 256 bits as there is also a record of which checkpoints the block is active in. The addresses in the leaves of the indexing trees for almost all files are not device addresses but are, instead, indexes into this array. The value found contains the actual device address. This allows a block to be relocated by simply moving the block and updating this file. All snapshots will immediately know where the new location is. Doing this with variable length extents would be impractical, which appears to be one of the main reasons that `NILFS2` doesn't use them. 

It should be noted that while this DAT file is similar to the NAT used by `f2fs`, it is different in scale. The `f2fs` NAT is used only to provide indirection when looking up nodes -- inodes, index blocks, etc. -- not when looking up data blocks. The DAT file is used for lookup of all blocks. This indirection imposes some cost on every access. 

> ![\[B-tree\]](https://static.lwn.net/images/2012/nilfs-b-tree.png)

Estimating that cost is, again, not easy. Given a 4K block size, each block in the DAT file provides indexing for 128 other blocks. This imposes approximately a 1% overhead in storage space and at least a 1% overhead in throughput. If all the DAT entries for a given file are adjacent, the overhead will be just that 1%. If they are very widely spread out, it could be as much as 100% (if each DAT entry is in a different block of the DAT file). Files that are created quickly on a fresh filesystem will tend to the smaller number, files created slowly (like log files) on a well-aged filesystem will likely tend toward a larger number. An average of 3% or 4% probably wouldn't be very surprising, but that is little more than a wild guess. 

Against this cost we must weigh the benefit, which is high frequency snapshots. While I have no experience with this feature, I do have experience with the "undo" feature in text editors. In my younger days I used "ed" and don't recall being upset by the lack of an undo feature. Today I use emacs and use undo all the time -- I don't know that I could go back to using an editor without this simple feature. I suspect continual snapshots are similar. I don't miss what I don't have, but I could quickly get used to them. 

So: is the availability of filesystem-undo worth a few percent in performance? This is a question I'll have to leave to my readers to sort out. To make it easier to ponder, I'll relieve your curiosity and clarify why not all files use the DAT layer of indirection. The answer is of course that the DAT file itself cannot use indirection as it would then be impossible to find anything. Every other file does use the DAT and every lookup in those files will involve indirection. 

#### Other miscellaneous metadata

`NILFS2` has two other metadata files. Both of these are simple tables of data without the allocation bitmaps of the DAT file and the `ifile`. 

The "`sufile`" records the usage of each segment of storage, counting the number of active blocks in the segment, and remembering when the segment was written. The former is used to allow the segment to be reused when it reaches zero. The latter is used to guide the choice of which segment to clean next. If a segment is very old, there is not much point waiting for more blocks to die of natural attrition. If it is young, it probably is worthwhile to wait a bit longer. 

The "`cpfile`" records checkpoints and every time a checkpoint is created a new record is added to the end of this file. This record stores enough information to reconstruct the state of all files at the time of the checkpoint. In particular, this includes the inode for the `ifile`. Left to itself, this file would grow without bound. However, in normal operation a user-space program ([nilfs_cleanerd](<http://www.nilfs.org/en/man8/nilfs_cleanerd.8.html>)) will monitor usage and delete old checkpoints as necessary. This results in the `cpfile` becoming a sparse file with lots of empty space for most of the length of the file, a dense collection of records at the end, and an arbitrary number of individual blocks sprinkled throughout (for the long-term snapshots). This is the file for which a radix-tree index may not be the optimal indexing scheme. It isn't clear that would matter though. 

#### Pros and cons

So now we must return to the question: from a technical perspective, would `NILFS2` make a suitable base of a flash-optimized filesystem? The principal property of flash, that the best writes are sequential writes aligned to the underlying erase block size, is easily met by `NILFS2`, making it a better contender than filesystems with lots of fixed locations, but can we gain any insight by looking at the details? 

One of the several flash-focused features of `f2fs` is that it has several segments open for writing at a time. This allows data with different life expectancies to be kept separate, and also improves the utilization of those flash devices that allow a degree of parallelism in access. `NILFS2` only has a single segment open at a time, as is probably appropriate for rotating media with a high seek cost, and makes no effort to sort blocks based on their life expectancy. Adding these to `NILFS2` would be possible, but it is unlikely that it would be straightforward. 

Looking at the more generally applicable features of `NILFS2`, the directory structure doesn't scale, the file indexing is less than optimal, and the addressing indirection imposes a cost of uncertain size. On the whole, there seems to be little to recommend it and a substantial amount of work that would be required to tune it to flash in the way that `f2fs` has been tuned. It gives the impression of being a one-big-idea filesystem. If you want continual snapshots, this is the filesystem for you. If not, it holds nothing of real interest. 

On the other side, `f2fs` comes across as a one-big-idea filesystem too. It is designed to interface with the FTL (Flash translation layer) found in today's flash devices, and provides little else of real interest, providing no snapshots and not working at all well with any storage device other than flash. Could this be a sign of a trend towards single-focus filesystems? And if so, is that a good thing, or a bad thing? 

So, while `NILFS2` could have been used as a starting point for a flash-focused filesystem, it is not at all clear that it would have been a good starting point, and it is hard to challenge the decision to create a new filesystem from scratch. Whether some other filesystem might have made a better start will have to be a question for another day.
