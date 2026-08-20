---
title: "Atomic block-write operations"
url: https://lwn.net/Articles/933015/
date: "May 30, 2023"
category: "Atomic IO operations; Block layer-Atomic operations"
author: "By Jake Edge May 30, 2023 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
May 30, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Martin Petersen and John Garry led a session at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>) on work they have been doing to implement atomic block writes of various sizes for SCSI and NVMe. The idea is to support devices that can guarantee atomic operations for sizes larger than their block size. It is an attempt to "find common ground" between the two standards, Petersen said, because the two have slightly different semantics, depending on the device type, and different restrictions, which has made for an "interesting project". It has been a challenge to find an abstraction layer that can work with the "five different variants of SCSI and NVMe implementations that may or may not be out there". 

Currently, they have a QEMU and a scsi_debug implementation of this work, Petersen continued. It is plumbed into [`pwritev2()`](<https://man7.org/linux/man-pages/man2/readv.2.html>) and io_uring, so that it can be used from applications. A special [`fallocate()`](<https://man7.org/linux/man-pages/man2/fallocate.2.html>) call can be made to tell the underlying filesystem that the application wants its file allocations to be aligned with whatever the hardware requires in order to provide atomic guarantees. An `fallocate2()` call was added to XFS for their testing. There is an interface for an application to query the hardware for the range of block sizes that it supports for atomic operations. The application can then do atomic block operations on the file using direct I/O. 

[ ![\[Martin Petersen\]](https://static.lwn.net/images/2023/lsfmb-petersen-sm.png) ](<https://lwn.net/Articles/933086/>)

Garry then described some more of the details. There is a new `RWF_ATOMIC` flag for `pwritev2()` that is part of the [patches](<https://lwn.net/ml/linux-fsdevel/20230503183821.1473305-1-john.g.garry@oracle.com/>) he posted in early May. He said that the patch developers come "from a database point of view", where the database has fixed block sizes, so the flag requests the kernel to write each database block atomically, not that the entire write (potentially consisting of multiple database blocks) is done atomically. 

The patches are "not too intrusive"; without counting the documentation, there are around 1200 lines of changes. About half of those changed lines are in `scsi_debug.c` because the locking model in that driver needed to change. There are about 300 changed lines in the block layer; atomic writes fit in the existing block layer pretty well, he said. There were also changes in the XFS code, which may or may not stay, he is not sure. 

Damien Le Moal asked why `fallocate()` was used rather than simply adding a direct I/O flag. Ted Ts'o said that the key is to ensure that the filesystem allocates the file data in a way that is aligned properly for the hardware. For ext4, that can already be done with a flag when the filesystem is created; it will then always allocate file data on the proper alignment boundaries. He fully supports the `fallocate()` approach and would add that to ext4 if it goes upstream; the advantage is that you do not have to create a specific filesystem in order to access the atomic capabilities of the device. 

Ts'o wondered about the need for the `pwritev2()` flag, however. His understanding of the NVMe spec is that devices advertise that they will not do partial writes (i.e. torn writes) for power-of-two sizes up to, say, 16KB or 32KB. So he was hoping for a simple change to the block layer to note that fact and not split BIOs (i.e. [`struct bio`](<https://elixir.bootlin.com/linux/v6.3.5/source/include/linux/blk_types.h#L252>) instances) at any other boundary. 

There are four new request-queue limits in the block layer, two of which are complementary, Garry said. There are unit minimum and maximum values, which are the smallest and largest size that are supported by the device for atomic writes; those are both powers of two and the expectation is that any block size used by applications will be as well. 

There is also an atomic write boundary value that is specific to NVMe; any I/O that crosses it will be split by the device. Petersen gave an example of a 128KB boundary value; any write that crosses the boundary that exists every 128KB will become two I/Os. That means that the block-allocation path needs to be careful about the boundary in order to avoid torn writes, he said. SCSI has its own boundary, but it is different Garry said. Fred Knight pointed out that as long as the I/Os start at the "right" place for the atomic-block alignment, they will not span the boundary. 

[ ![\[John Garry\]](https://static.lwn.net/images/2023/lsfmb-garry-sm.png) ](<https://lwn.net/Articles/933086/>)

The fourth value "is max bytes, just to confuse things", Garry said. It may be different than the unit maximum; it specifies the total maximum size for an I/O consisting of atomic-write operations. Petersen said that the overall I/O might consist of, say, 16 chunks, each of which must be written atomically. It is the unit minimum and maximum that user-space applications need to be aware of, Garry said; those can be retrieved using [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>). The other two are only used internally to the block layer. 

Ts'o said he had mostly ignored those parameters because the database developers do not seem to care about anything other than 16KB atomic writes. The database may send some large number of those 16KB chunks in a single write, but only need to be guaranteed that those chunks are not torn; the whole I/O can be torn on those boundaries without a problem. Hearkening back to his [talk](<https://lwn.net/Articles/932900/>), he said that the cloud providers could simply have their block devices support the 16KB requirement, without the extras, though he understands why a more general solution might be needed for other use cases. 

In response to a question from Jan Kara, Garry said that user-space does not explicitly choose an atomic-block size. The atomic-block size is inferred by the block layer from the alignment and size of the write. Kara is concerned about partition alignment and device-mapper interactions that will interfere with that and wondered if some kind of offset will be needed from the user-space side, but Petersen said that these partition-alignment problems have already been solved for other reasons; the intent is to keep things simple. 

Javier González asked about the range of sizes being considered in the work; there are limits at various levels, so what use cases are being targeted. Petersen said that database systems generally have 8KB, 16KB, or 32KB blocks and typically do their writes in chunks of 512KB to 1MB, which is what they want to facilitate. Le Moal said it will probably be difficult for the devices to support much more than that. 

Garry said that once the block layer has inferred the block size for the application, it uses that whenever a write is done. It fills BIOs to that size or a multiple of it; when BIOs are split, the inferred block size is used. Kara wondered about what happens if user space submits unaligned or incorrect-length writes; Garry said that the code does rely on "careful user-space programming". Knight said that one of the differences between NVMe and SCSI is that NVMe will simply perform that kind of write non-atomically, while SCSI returns an error. 

Ts'o said that he understood why the initial implementation is only for direct I/O, but he would like to find a way to support PostgreSQL, which uses buffered I/O. He is hoping there could be some way to teach the writeback code that some set of contiguous page-cache pages correspond to a user-space block that should be written atomically. 

Petersen asked Darrick Wong, who was dialed in remotely, if he had thoughts on how to make that work. Wong said that he was unsure how to do atomic writes for page-cache pages, but thought perhaps there could be some kind of mode that indicated that a file should only be written with atomic writes, "then try to do it right". He does not think it would be impossible to do using the iomap interface, but it "would be a pain" because the folio sizes and the atomic-block sizes may not be the same. 

Bart Van Assche suggested raising the overall block size to 16KB, so it is guaranteed that the writes are aligned and are a multiple of that size. If it is necessary to do smaller writes, sub-block write operations could be added. Ts'o thought that the block size could be set on a per-inode basis using [`fcntl()`](<https://man7.org/linux/man-pages/man2/fcntl.2.html>), then the writeback code would know to do atomic writes on properly aligned and sized sets of dirty pages; pages that were not aligned and appropriately sized would get no guarantee with regard to atomicity. It would be somewhat fragile and would not be as good as the direct I/O implementation, he said, but would not require any code changes for PostgreSQL to take advantage of it. 

Wong said that perhaps XFS could support 16KB blocks; for years, Dave Chinner has been heard to mutter that the filesystem is "really really close" to being able to do so. It mostly requires changes to iomap to handle multiple pages in a block and then fixing the size of folios to 16KB, Wong said. 

Luis Chamberlain said that the work that is going on to support larger block sizes should sort out what is needed to support atomic writes for buffered I/O. He would be leading a discussion on that topic the next day and thinks that a good outcome from LSFMM would be to flesh out the different use cases and to come up with test cases for all of them. His main concern is for memory fragmentation if the underlying folios are not being created and freed at the same rate. González thought that the atomic-block size would generally be the same throughout the system, but Petersen said that there are common use cases where the database-block size is different between databases on the same filesystem. 

The conversation continued on for a ways, going in several different directions. The feature is fairly small and works now for direct I/O, Petersen said; certainly people in the room were interested in seeing it in the kernel and had plenty of ideas for where it could go from there.
