---
title: "An update on torn-write protection"
url: https://lwn.net/Articles/1016406/
date: "April 9, 2025"
category: "Atomic IO operations; Filesystems-XFS"
author: "By Jake Edge April 9, 2025 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 9, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

In a combined storage and filesystem track session at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit, John Garry continued the theme of "untorn" (or atomic) writes that started in [the previous session](<https://lwn.net/Articles/1016015/>). It was also an update on where things have gone for untorn writes since his [session at last year's summit](<https://lwn.net/Articles/974578/>). Beyond that, he looked at some of the plans and challenges for the feature in the future. 

[ ![\[John Garry\]](https://static.lwn.net/images/2025/lsfmb-garry-sm.png) ](<https://lwn.net/Articles/1016880/>)

Garry called the feature, which he has been working on for a year or two at this point, "torn-write protection". The idea is to prevent the system from "tearing" a write operation by only writing part of it. It is important for database systems, he said, so that they do not need to double-buffer their data to protect against partial writes when there is a power failure or system crash. 

The `RWF_ATOMIC` flag has been added to the [`pwritev2()`](<https://www.man7.org/linux/man-pages/man2/pwritev.2.html>) system call to specify that a given write should either be committed in full to the storage device—or that none of it should be. But, in order to guarantee persistence, `RWF_SYNC` (or similar operations) are still required. The supported minimum and maximum sizes for atomic writes can be queried using the `STATX_WRITE_ATOMIC` flag for the [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) system call. Those values will be powers of two and any `RWF_ATOMIC` writes will need to also have a power of two length value between the minimum and maximum; the buffer will need to be aligned, as well. 

Chuck Lever could ""hear the eyeballs rolling in the back of the room"" but wondered about databases that use network filesystems for their storage; he does not think that untorn writes are supported there. The Linux NFS client supports direct I/O, at the request of database developers, he said, but it would require support in the NFS protocol to handle untorn writes. Garry said that the iomap layer sets a flag on untorn writes that the block layer can use, but Lever pointed out that the block layer of interest would be on the server, so the two of them agreed that some work will need to be done to support that. 

#### Hardware based

Garry said that SCSI and NVMe have non-complementary feature sets. For NVMe, all writes are implicitly atomic as there is no dedicated command to request an atomic write, unlike SCSI, which has the `WRITE_ATOMIC_16` command for that purpose. But NVMe writes are only actually atomic if they follow the rules on write lengths and do not cross an atomic-write boundary if one exists. That's not great, he said, because there is no indication if those rules are broken; the write could end up torn. The Linux NVMe driver detects that condition and returns an error, however. SCSI will reject the atomic command if its conditions are not met. 

The virtual filesystem (VFS) and block-layer support for untorn writes on SCSI and NVMe were merged into the 6.12 kernel. There is also [support for XFS](<https://lwn.net/Articles/1009298/>), but it is limited to writes with a size of a single filesystem block. At roughly the same time, though, support for [filesystem blocks larger than the page size](<https://lwn.net/Articles/1009548/>) was merged, so XFS filesystems with an 8KB or 16KB block size can do untorn writes for those sizes using direct I/O. 

Due to the way the iomap layer works, an atomic write cannot currently be done for a mixed range, such as a range containing both data and holes; that could be solved, but it is would be painful to do, he said. A bigger problem for XFS is that there is no guarantee that the disk blocks in an atomic write are contiguous or that they are properly aligned. The filesystem blocks could be ""backed by disk blocks that are sparsely spread out through the filesystem"". 

He and others had been pushing the addition of an XFS forcealign attribute as a solution to those problems. It would guarantee that filesystem blocks were allocated and aligned correctly. The [XFS realtime device](<https://blogs.oracle.com/linux/post/xfs-realtime-device>) has "large allocation units" that can be used to provide the needed guarantees, so forcealign extended that feature to the regular filesystem, but other XFS developers did not seem to think that was the right thing to do. The forcealign feature is, thus, not being pursued currently. 

So he turned to the large-block-size work. Switching the filesystem block size to 16KB would inherently provide the needed alignment guarantees and XFS already supported writing a single filesystem block atomically. But, when testing MySQL performance using a 16KB filesystem block (which is the same as the database block size) and atomic writes, he and his colleagues found ""significant performance impact"" in some tests, particularly those with a lower number of threads. Using double-buffered writes for the database performed better than atomic writes. 

The problem was diagnosed to be from writes of the "[redo log](<https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html>)", which is a buffered 512-byte write followed by an [`fsync()`](<https://www.man7.org/linux/man-pages/man2/fsync.2.html>). With a 4KB filesystem block, that is inefficient because it is updating much less than the block, but raising the block size to 16KB only makes that worse. There have been efforts to improve the performance of the redo log, but this kind of pattern is seen in lots of different kinds of applications; it is not just a MySQL problem. 

#### Filesystem based

So far, only the hardware-based solutions for atomic writes have been pursued, Garry said. A while back, Christoph Hellwig worked on [atomic writes for filesystems](<https://lwn.net/Articles/789600/>), which was (originally) based on opening files using an `O_ATOMIC` flag. Filesystem-based atomic writes would not have the alignment and single-extent requirements that come with the hardware-based variant. In addition, since hardware with atomic-write support is uncommon, a filesystem-based variant would bring the feature to many more users. 

So he is currently working on XFS atomic writes. A write of that kind would allocate staging blocks where the data gets written, non-atomically, and then the block mappings will be updated atomically in a single transaction. The XFS copy-on-write (CoW) [fork](<https://en.wikipedia.org/wiki/Fork_\(file_system\)>) can be used for the staging blocks, but it will mean that a write requires an allocation, block remapping, and a free operation, so it will be slower. Unlike the hardware-based solution, though, it would not require a reformatting of the filesystem to increase the block size as long as the existing filesystem supports XFS reflink. 

When the CoW blocks are allocated, aligned blocks are requested so that hardware-based atomic writes can be used if that is available. It would be a hybrid approach, that would first try to do the atomic write via the hardware. If the alignment or write-size are not suitable, it would fall back to the filesystem-based atomic writes. 

Amir Goldstein asked if user space can test for which mode it will get or request that only a particular mode is used. Is there a way for the application to know which will be used, he wondered. The idea is that once the database, for example, has been running for a while, everything will be aligned and all of the atomic writes will be done with the hardware, Garry said. 

Mike Snitzer noted that it made sense to do this work for XFS, since it is widely used, but he is concerned that the work is XFS-specific. He wondered if the feature would be useful for any filesystem, including NFS and other network filesystems, returning to Lever's question. Is there a plan for a more generic mechanism to join multiple blocks into an atomic write? Garry said that there are features like "bigalloc" for ext4 that can be used for that filesystem; that work is currently ongoing. But Snitzer said that was just another filesystem-specific scheme and not something generic at the virtual filesystem (VFS) layer that any Linux filesystem might be able to use. 

SCSI and NVMe support for atomic writes requires that the blocks involved in the write be contiguous on the storage device, Ted Ts'o said, which is ultimately a filesystem-specific attribute. He has chosen not to use the forcealign approach for ext4 because it would require that the database files be restored onto a new filesystem, with different attributes and less fragmentation, which is not popular for production databases. 

He has funding from his company (Google) to support untorn 16KB writes on ext4 for databases, but nothing further than that. He hopes to have that support get merged as part of the Linux 6.16 release. It uses the bigalloc feature with a 16KB cluster size in order to get the required alignment for the underlying hardware. There are vendors ""shipping product today, they're just simply relying on the fact that 'we desk-checked the block layer'"" and that the vendors' testing has shown that in practice writes will not be torn. ""Yes, this is terrifying, this is why I want all of this stuff to land for real."" 

Chris Mason returned to Goldstein's point, noting that silently going from the fast, hardware-based atomic writes to slow, filesystem-based writes is not what his customers want. They want to use the hardware-based mechanism and to get an error if that cannot be done. Garry said that in the testing that has been done, the software-based writes do not ""typically"" occur. 

But Mason said that means that in production once in a while, writes will start being slow; he would rather get an error. Garry said that is not a good user experience, however. Josef Bacik agreed with Mason, saying that something that randomly slows down once in a while will cause him to ""lose my mind and we just won't use it""; he noted that "typically" means that ""across 100,000 nodes it happens once a day on a random machine"". 

Garry asked what the user is supposed to do if they get an error instead. Bacik and Mason said that it will then be clear that something in the environment was misconfigured or otherwise broken so that it can be addressed. Hellwig said that the developers should fix the instrumentation of the system ""instead of creating stupid interfaces"". The information on whether the fallback has been used is easily available from a tracepoint, but Mason pointed out that his users cannot run all of the systems with tracepoints enabled. 

Ts'o said that even with bigalloc, there are ways that users could mess up the atomic-write requirements; for example, by punching holes in every other block in the cluster. Ext4 will do the right thing, he said, by writing zeroes into the holes and fixing up the extent trees so that atomic writes can be submitted to the block layer, but ""will log a message saying 'user space did a stupid'"". That is so he can close bugs with performance complaints when that happens; he could do it with tracepoints, but those are generally not enabled in production. 

Garry closed by mentioning a [`statx()` change](<https://lwn.net/ml/linux-xfs/20250319114402.3757248-1-john.g.garry@oracle.com/>) he would like to make to report the maximum length for the hardware atomic write. That would allow applications to try to ensure their atomic writes go as fast as they can without falling back to the slower filesystem-based version. His last item was the upstream status of the work. The iomap changes for the filesystem-based atomic writes were submitted for 6.15 and the XFS support for the feature is targeted for 6.16.
