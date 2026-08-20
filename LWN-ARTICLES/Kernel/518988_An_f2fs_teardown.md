---
title: An f2fs teardown
url: https://lwn.net/Articles/518988/
date: "October 10, 2012"
category: "Filesystems-Flash"
author: "October 10, 2012 This article was contributed by Neil Brown"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

October 10, 2012

This article was contributed by Neil Brown

When a techno-geek gets a new toy there must always be an urge to take it apart and see how it works. Practicalities (and warranties) sometimes suppress that urge, but in the case of `f2fs` and this geek, the urge was too strong. What follows is the result of taking apart this new filesystem to see how it works. 

`f2fs` (interestingly not "`f3s`") is the "flash-friendly file system", a new filesystem for Linux recently [announced](<https://lwn.net/Articles/518717/>) by engineers from Samsung. Unlike `jffs2` and `logfs`, `f2fs` is not targeted at raw flash devices, but rather at the specific hardware that is commonly available to consumers -- SSDs, eMMC, SD cards, and other flash storage with an [FTL](<http://en.wikipedia.org/wiki/Flash_file_system#Translation_layers>) (flash translation layer) already built in. It seems that as hardware gets smarter, we need to make even more clever software to manage that "smartness". Does this sound like parenting to anyone else? 

`f2fs` is based on the log-structured filesystem (LFS) design -- which is hardly surprising given the [close match](<https://lwn.net/Articles/353411/>) between the log-structuring approach and the needs of flash. For those not familiar with log-structured design, the key elements are: 

  1. That it requires copy-on-write, so data is always written to previously unused space.

  2. That free space is managed in large regions which are written to sequentially. When the number of free regions gets low, data that is still live is coalesced from several regions into one free region, thus creating more free regions. This process is known as "cleaning" and the overhead it causes is one of the significant costs of log structuring.

As the FTL typically uses a log-structured design to provide the wear-leveling and write-gathering that flash requires, this means that there are two log structures active on the device -- one in the firmware and one in the operating system. `f2fs` is explicitly designed to make use of this fact and leaves a number of tasks to the FTL while focusing primarily on those tasks that it is well positioned to perform. So, for example, `f2fs` makes no effort to distribute writes evenly across the address space to provide wear-leveling. 

The particular value that `f2fs` brings, which can justify it being "flash friendly", is that it provides large-scale write gathering so that when lots of blocks need to be written at the same time they are collected into large sequential writes which are much easier for the FTL to handle. Rather than creating a single large write, `f2fs` actually creates up to six in parallel. As we shall see, these are assigned different sorts of blocks with different life expectancies. Grouping blocks with similar life expectancies together tends to make the garbage collection process required by the LFS less expensive. 

The "large-scale" is a significant qualifier -- `f2fs` doesn't always gather writes into contiguous streams, only almost always. Some metadata, and occasionally even some regular data, is written via random single-block writes. This would be anathema for a regular log-structured filesystem, but `f2fs` chooses to avoid a lot of complexity by just doing small updates when necessary and leaving the FTL to make those corner cases work. 

Before getting into the details of how `f2fs` does what it does, a brief list of some of the things it doesn't do is in order. 

A feature that we might expect from a copy-on-write filesystem is cheap snapshots as they can be achieved by simply not freeing up the old copy. `f2fs` does not provide these and cannot in its current form due to its two-locations approach to some metadata which will be detailed later. 

Other features that are missing are usage quotas, NFS export, and the "security" flavor of extended attributes (xattrs). Each of these could probably be added with minimal effort if they are needed, though integrating quotas correctly with the crash recovery would be the most challenging. We shouldn't be surprised to see some of these in a future release. 

#### Blocks, segments, sections, and zones

Like most filesystems, `f2fs` is comprised of blocks. All blocks are 4K in size, though the code implicitly links the block size with the system page size, so it is unlikely to work on systems with larger page sizes as is possible with IA64 and PowerPC. The block addresses are 32 bits so the total number of addressable bytes in the filesystem is at most 2(32+12) bytes or 16 terabytes. This is probably not a limitation -- for current flash hardware at least. 

Blocks are collected into "segments". A segment is 512 blocks or 2MB in size. The documentation describes this as a default, but this size is fairly deeply embedded in the code. Each segment has a segment summary block which lists the owner (file plus offset) of each block in the segment. The summary is primarily used when cleaning to determine which blocks need to be relocated and how to update the index information after the relocation. One block can comfortably store summary information for 512 blocks (with a bit of extra space which has other uses), so 2MB is the natural size for a segment. Larger would be impractical and smaller would be wasteful. 

Segments are collected into sections. There is genuine flexibility in the size of a section, though it must be a power of two. A section corresponds to a "region" in the outline of log structuring given above. A section is normally filled from start to end before looking around for another section, and the cleaner processes one section at a time. The default size when using the `mkfs` utility is 20, or one segment per section. 

`f2fs` has six sections "open" for writing at any time with different sorts of data being written to each one. The different sections allows for file content (data) to be kept separate from indexing information (nodes), and for those to be divided into "hot", "warm", and "cold" according to various heuristics. For example, directory data is treated as hot and kept separate from file data because they have different life expectancies. Data that is cold is expected to remain unchanged for quite a long time, so a section full of cold blocks is likely to not require any cleaning. Nodes that are hot are expected to be updated soon, so if we wait a little while, a section that was full of hot nodes will have very few blocks that are still live and thus will be cheap to clean. 

Sections are collected into zones. There may be any (integer) number of sections in a zone though the default is again one. The sole purpose of zones is to try to keep these six open sections in different parts of the device. The theory seems to be that flash devices are often made from a number of fairly separate sub-devices each of which can process IO requests independently and hence in parallel. If zones are sized to line up with the sub-devices, then the six open sections can all handle writes in parallel and make best use of the device. 

These zones, full of sections of segments of blocks, make up the "main" area of the filesystem. There is also a "meta" area which contains a variety of different metadata such as the segment summary blocks already mentioned. This area is not managed following normal log-structured lines and so leaves more work for the FTL to do. Hopefully it is small enough that this isn't a problem. 

There are three approaches to management of writes in this area. First, there is a small amount of read-only data (the superblock) which is never written once the filesystem has been created. Second, there are the segment summary blocks which have already been mentioned. These are simply updated in-place. This can lead to uncertainty as to the "correct" contents for the block after a crash, however for segment summaries this is not an actual problem. The information in it is checked for validity before it is used, and if there is any chance that information is missing, it will be recovered from other sources during the recovery process. 

The third approach involves allocating twice as much space as is required so that each block has two different locations it can exist in, a primary and a secondary. Only one of these is "live" at any time and the copy-on-write requirement of an LFS is met by simply writing to the non-live location and updating the record of which is live. This approach to metadata is the main impediment to providing snapshots. `f2fs` does a small amount of journaling of updates to this last group while creating a checkpoint, which might ease the task for the FTL somewhat. 

#### Files, inodes, and indexing

Most modern filesystems seem to use B-trees or similar structures for managing indexes to locate the blocks in a file. In fact they are so fundamental to `btrfs` that it takes its name from that data structure. `f2fs` doesn't. Many filesystems reduce the size of the index by the use of "extents" which provide a start and length of a contiguous list of blocks rather than listing all the addresses explicitly. Again, `f2fs` doesn't (though it does maintain one extent per inode as a hint). 

Rather, `f2fs` uses an indexing tree that is very reminiscent of the original Unix filesystem and descendants such as ext3. The inode contains a list of addresses for the early blocks in the file, then some addresses for indirect blocks (which themselves contain more addresses) as well as some double and triple-indirect blocks. While ext3 has 12 direct addresses and one each of the indirection addresses, `f2fs` has 929 direct address, two each of indirect and double-indirect addresses, and a single triple-indirect address. This allows the addressing of nearly 4TB for a file, or one-quarter of the maximum filesystem size. 

While this scheme has some costs -- which is why other filesystems have discarded it -- it has a real benefit for an LFS. As `f2fs` does not use extents, the index tree for a given file has a fixed and known size. This means that when blocks are relocated through cleaning, it is impossible for changes in available extents to cause the indexing tree to get bigger -- which could be embarrassing when the point of cleaning is to free space. `logfs`, another reasonably modern log structured filesystem for flash, uses much the same arrangement for much the same [reason](<http://lca2007.linux.org.au/talk/91.html>). 

Obviously, all this requires a slightly larger inode than `ext3` uses. Copy-on-write is rather awkward for objects that are smaller than the block size so `f2fs` reserves a full 4K block for each inode which provides plenty of space for indexing. It even provides space to store the (base) name of the file, or one of its names, together with the inode number of the parent. This simplifies the recovery of recently-created files during crash recovery and reduces the number of blocks that need to be written for such a file to be safe. 

Given that the inode is so large, one would expect that small files and certainly small symlinks would be stored directly in the inode, rather than just storing a single block address and storing the data elsewhere. However `f2fs` doesn't do that. Most likely the reality is that it doesn't do it _yet_. It is an easy enough optimization to add, so it's unlikely to remain absent for long. 

As already mentioned, the inode contains a single extent that is a summary of some part of the index tree. It says that some range of blocks in the file are contiguous in storage and gives the address of this range. The filesystem attempts to keep the largest extent recorded here and uses it to speed up address lookups. For the common case of a file being written sequentially without any significant pause, this should result in the entire file being in that one extent, and make lookups in the index tree unnecessary. 

Surprisingly, it doesn't seem there was enough space to store 64-bit timestamps, so instead of nanosecond resolution for several centuries in the future, it only provides single-second resolution until some time in 2038. This oversight was [raised on linux-kernel](<https://lwn.net/Articles/519032/>) and may well be addressed in a future release. 

One of the awkward details of any copy-on-write filesystem is that whenever a block is written, its address is changed, so its parent in the indexing tree must change and be relocated, and so on up to the root of the tree. The logging nature of an LFS means that roll-forward during recovery can rebuild recent changes to the indexing tree so all the changes do not have to be written immediately, but they do have to be written eventually, and this just makes more work for the cleaner. 

This is another area when `f2fs` makes use of its underlying FTL and takes a short-cut. Among the contents of the "meta" area is a NAT -- a Node Address Table. Here "node" refers to inodes and to indirect indexing blocks, as well as blocks used for xattr storage. When the address of an inode is stored in a directory, or an index block is stored in an inode or another index block, it isn't the block address that is stored, but rather an offset into the NAT. The actual block address is stored in the NAT at that offset. This means that when a data block is written, we still need to update and write the node that points to it. But writing that node only requires updating the NAT entry. The NAT is part of the metadata that uses two-location journaling (thus depending on the FTL for write-gathering) and so does not require further indexing. 

#### Directories

An LFS doesn't really impose any particular requirements on the layout of a directory, except to change the fewest number of blocks possible, which is generally good for performance anyway. So we can assess `f2fs`'s directory structure on an equal footing with other filesystems. The primary goal is to provide fast lookup by file name, and to provide a stable address of each name that can be reported using `telldir()`. 

The original Unix filesystem (once it had been adjusted for 256-byte file names) used the same directory scheme as `ext2` -- sequential search though a file full of directory entries. This is simple and effective, but doesn't scale well to large directories. 

More modern filesystems such as `ext3`, `xfs`, and `btrfs` use various schemes involving B-trees, sometimes indexed by a hash of the file name. One of the problems with B-trees is that nodes sometimes need to be split and this causes some directory entries to be moved around in the file. This results in extra challenges to provide stable addresses for `telldir()` and is probably the reason that `telldir()` is often called out for being a poor interface. 

`f2fs` uses some sequential searching and some hashing to provide a scheme that is simple, reasonably efficient, and trivially provides stable `telldir()` addresses. A lot of the hashing code is borrowed from ext3, however f2fs omits the use of a per-directory seed. This seed is a secret random number which ensures that the hash values used are different in each directory, so they are not predictable. Using such a seed provides protection against hash-collision attacks. While these might be unlikely in practice, they are so easy to prevent that this omission is a little surprising. 

It is easiest to think of the directory structure as a series of hash tables stored consecutively in a file. Each hash table has a number of fairly large buckets. A lookup proceeds from the first hash table to the next, at each stage performing a linear search through the appropriate bucket, until either the name is found or the last hash table has been searched. During the search, any free space in a suitable bucket is recorded in case we need to create the name. 

The first hash table has exactly one bucket which is two blocks in size, so for the first few hundred entries, a simple linear search is used. The second hash table has two buckets, then four, then eight and so on until the 31st table with about a billion buckets, each two blocks in size. Subsequent hash tables -- should you need that many -- all have the same number of buckets as the 31st, but now they are four blocks in size. 

The result is that a linear search of several hundred entries can be required, possibly progressing through quite a few blocks if the directory is very large. The length of this search increases only as the logarithm of the number of entries in the directory, so it scales fairly well. This is certainly better than a purely sequential search, but seems like it could be a lot more work than is really necessary. It does however guarantee that only one block needs to be updated for each addition or deletion of a file name, and since entries are never moved, the offset in the file is a stable address for `telldir()`, which are valuable features. 

#### Superblocks, checkpoints, and other metadata

All filesystems have a superblock and `f2fs` is no different. However it does make a clear distinction between those parts of the superblock which are read-only and those which can change. These are kept in two separate data structures. 

The `f2fs_super_block`, which is stored in the second block of the device, contains only read-only data. Once the filesystem is created, this is never changed. It describes how big the filesystem is, how big the segments, sections, and zones are, how much space has been allocated for the various parts of the "meta" area, and other little details. 

The rest of the information that you might expect to find in a superblock, such as the amount of free space, the address of the segments that should be written to next, and various other volatile details, are stored in an `f2fs_checkpoint`. This "checkpoint" is one of the metadata types that follows the two-location approach to copy-on-write -- there are two adjacent segments both of which store a checkpoint, only one of which is current. The checkpoint contains a version number so that when the filesystem is mounted, both can be read and the one with the higher version number is taken as the live version. 

We have already mentioned the Node Address Table (NAT) and Segment Summary Area (SSA) that also occupy the meta area with the superblock (SB) and Checkpoints (CP). The one other item of metadata is the Segment Info Table or SIT. 

The SIT stores 74 bytes per segment and is kept separate from the segment summaries because it is much more volatile. It primarily keeps track of which blocks are still in active use so that the segment can be reused when it has no active blocks, or can be cleaned when the active block count gets low. 

When updates are required to the NAT or the SIT, `f2fs` doesn't make them immediately, but stores them in memory until the next checkpoint is written. If there are relatively few updates then they are not written out to their final home but are instead journaled in some spare space in Segment Summary blocks that are normally written at the same time. If the total amount of updates that are required to Segment Summary blocks is sufficiently small, even they are not written and the SIT, NAT, and SSA updates are all journaled with the Checkpoint block -- which is always written during checkpoint. Thus, while `f2fs` feels free to leave some work to the FTL, it tries to be friendly and only performs random block updates when it really has to. When `f2fs` does need to perform random block updates it will perform several of them at once, which might ease the burden on the FTL a little. 

#### Knowing when to give up

Handling filesystem-full conditions in traditional filesystems is relatively easy. If no space is left, you just return an error. With a log-structured filesystem, it isn't that easy. There might be a lot of free space, but it might all be in different sections and so it cannot be used until those sections are "cleaned", with the live data packed more densely into fewer sections. It usually makes sense to over-provision a log-structured filesystem so there are always free sections to copy data to for cleaning. 

The FTL takes exactly this approach and will over-provision to both allow for cleaning and to allow for parts of the device failing due to excessive wear. As the FTL handles over-provisioning internally there is little point in `f2fs` doing it as well. So when `f2fs` starts running out of space, it essentially gives up on the whole log-structured idea and just writes randomly wherever it can. Inodes and index blocks are still handled carefully and there is a small amount of over-provisioning for them, but data is just updated in place, or written to any free block that can be found. Thus you can expect performance of `f2fs` to degrade when the filesystem gets close to full, but that is common to a lot of filesystems so it isn't a big surprise. 

#### Would I buy one?

`f2fs` certainly seems to contain a number of interesting ideas, and a number of areas for possible improvement -- both attractive attributes. Whether reality will match the promise remains to be seen. One area of difficulty is that the shape of an `f2fs` (such as section and zone size) needs to be tuned to the particular flash device and its FTL; vendors are notoriously secretive about exactly how their FTL works. f2fs also requires that the flash device is comfortable having six or more concurrently "open" write areas. This may not be a problem for Samsung, but does present some problems for your average techno-geek -- though Arnd Bergmann has [done some research](<https://lwn.net/Articles/428584/>) that may prove useful. If this leads to people reporting performance results based on experiments where the `f2fs` isn't tuned properly to the storage device, it could be harmful for the project as a whole. 

`f2fs` contains a number of optimizations which aim to ease the burden on the FTL. It would be very helpful to know how often these actually result in a reduction in the number of writes. That would help confirm that they are a good idea, or suggest that further refinement is needed. So, some gathering of statistics about how often the various optimizations fire would help increase confidence in the filesystem. 

`f2fs` seems the have been written without much expectation of highly parallel workloads. In particular, all submission of write requests are performed under a single semaphore. So `f2fs` probably isn't the filesystem to use for big-data processing on 256-core work-horses. It should be fine on mobile computing devices for a few more years though. 

And finally, lots of testing is required. Some preliminary performance measurements have been [posted](<https://lwn.net/Articles/519033/>), but to get a fair comparison you really need an "aged" filesystem and a large mix of workloads. Hopefully someone will make the time to do the testing. 

Meanwhile, would I use it? Given that my phone is as much a toy to play with as a tool to use, I suspect that I would. However, I would make sure I had reliable backups first. But then ... I probably should do that anyway.
