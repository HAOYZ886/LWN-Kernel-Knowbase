---
title: Fast commits for ext4
url: https://lwn.net/Articles/842385/
date: "January 15, 2021"
category: "Filesystems-ext4"
author: "January 15, 2021 This article was contributed by Marta Rybczyńska"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

January 15, 2021

This article was contributed by Marta Rybczyńska

The Linux 5.10 release included a [change](<https://lwn.net/ml/linux-ext4/20201015203802.3597742-1-harshadshirwadkar@gmail.com/>) that is expected to significantly increase the performance of the ext4 filesystem; it goes by the name "fast commits" and introduces a new, lighter-weight journaling method. Let us look into how the feature works, who can benefit from it, and when its use may be appropriate.

Ext4 is a [journaling filesystem](<https://en.wikipedia.org/wiki/Journaling_file_system>), designed to ensure that filesystem structures appear consistent on disk at all times. A single filesystem operation (from the user's point of view) may require multiple changes in the filesystem, which will only be coherent after all of those changes are present on the disk. If a power failure or a system crash happens in the middle of those operations, corruption of the data and filesystem structure (including unrelated files) is possible. Journaling prevents corruption by maintaining a log of transactions in a separate journal on disk. In case of a power failure, the recovery procedure can replay the journal and restore the filesystem to a consistent state.

The ext4 journal includes the metadata changes associated with an operation, but not necessarily the related data changes. Mount options can be used to select one of three journaling modes, as described in the [ext4 kernel documentation](<https://www.kernel.org/doc/html/latest/admin-guide/ext4.html>). `data=ordered`, the default, causes ext4 to write all data _before_ committing the associated metadata to the journal. It [does not put the data itself into the journal](<https://lwn.net/Articles/322823/>). The `data=journal` option, instead, causes all data to be written to the journal before it is put into the main filesystem; as a side effect, it disables delayed allocation and direct-I/O support. Finally, `data=writeback` relaxes the constraints, allowing data to be written to the filesystem after the metadata has been committed to the journal.

Another important ext4 feature is delayed allocation, where the filesystem defers the allocation of blocks on disk for data written by applications until that data is actually written to disk. The idea is to wait until the application finishes its operations on the file, then allocate the actual number of data blocks needed on the disk at once. This optimization limits unneeded operations related to short-lived, small files, batches large writes, and helps ensure that data space is allocated contiguously. On the other hand, the writing of data to disk might be delayed (with the default settings) by a minute or so. In the default `data=ordered` mode, where the journal entry is written only after flushing all pending data, delayed allocation might thus add more delay between the actual change and writing of the journal. To assure data is actually written to disk, applications use the [`fsync()` or `fdatasync()`](<https://man7.org/linux/man-pages/man2/fsync.2.html>) system calls, causing the data (and the journal) to be written immediately. 

#### Ext4 journal optimization

One might assume that, in such a situation, there are a number of optimizations that could be made in the commit path; that assumption turns out to be correct. In [this USENIX'17 paper [PDF]](<https://www.usenix.org/system/files/conference/atc17/atc17-park.pdf>), Daejun Park and Dongkun Shin showed that the current ext4 journaling scheme can introduce significant latencies because `fsync()` causes a lot of unrelated I/O. They proposed a faster scheme, taking into account the fact that some of the metadata written to the journal could instead be derived from changes to the inode being written, and it is possible to commit transactions related to the requested file descriptor only. Their optimization works in the `data=ordered` mode.

The fast-commit changes, implemented by Harshad Shirwadkar, are based on the work of Park and Shin. This work implements an additional journal for fast commits, but simplifies the commit path. There are now two journals in the filesystem: the fast-commit journal for operations that can be optimized, and the regular journal for "standard commits" whose handling is unchanged. The fast-commit journal contains operations executed since the last standard commit.

Ext4 uses a generic journaling layer called "Journaling Block Device 2" (JBD2), with the exact on-disk format documented in the [ext4 wiki](<https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#Journal>). JBD2 operates on blocks, so when it commits a transaction, this transaction includes all changed blocks. One logical change may affect multiple blocks, for example the inode table and the block bitmap.

The fast-commit journal, on the other hand, contains changes at the file level, resulting in a more compact format. Information that can be recreated is left out, as described in the patch posting: 

For example, if a new extent is added to an inode, then corresponding updates to the inode table, the block bitmap, the group descriptor and the superblock can be derived based on just the extent information and the corresponding inode information. 

During recovery from this journal, the filesystem must recalculate all changed blocks from the inode changes, and modify all affected data structures on the disk. This requires specific code paths for each file operation, and not all of them are implemented right now. The fast-commits feature currently supports unlinking and linking a directory entry, creating an inode and a directory entry, adding blocks to and removing blocks from an inode, and recording an inode that should be replayed.

Fast commits are an addition to — not a replacement of — the standard commit path; the two work together. If fast commits cannot handle an operation, the filesystem falls back to the standard commit path. This happens, for example, for changes to extended attributes. During recovery, JBD2 first performs replay of the standard transactions, then lets the filesystem recover fast commits.

#### `fsync()` side effects

The fast-commit optimization is designed to work with applications using `fsync()` frequently to ensure data integrity. When we look at the `fsync()` and `fdatasync()` man pages, we see that those system calls only guarantee to write data linked to the given file descriptor. With ext4, as a side effect of the filesystem structure, all pending data and metadata for _all_ file descriptors will be flushed instead. This creates a lot of I/O traffic that is unneeded to satisfy any given `fsync()` or `fdatasync()` call. 

This side effect leads to a difference between the paper and the implementation: a fast commit may still include changes affecting other files. In a review, Jan Kara [asked](<https://lwn.net/ml/linux-ext4/20201023103013.GF25702@quack2.suse.cz/>) why unrelated changes are committed. Shirwadkar [replied](<https://lwn.net/ml/linux-ext4/CAD+ocbws2J0boxfNA+gahWwTAqm8-Pef9_WkcwwKFjpiJhvJKw@mail.gmail.com/>) that, in an earlier version of the patch, he did indeed write only the file in question. However, this change broke some existing tests that depend on `fsync()` working as a global barrier, so he backed it out. 

Ted Ts'o [commented](<https://lwn.net/ml/linux-ext4/20201027184507.GD5691@mit.edu/>) that the current version of the patch set keeps the existing behavior, but he can see workloads where ""not requiring entanglement of unrelated file writes via fsync(2) could be a huge performance win."" He added that a future solution could be a new system call taking an array of file descriptors to synchronize together. For now, application developers should base their code on the POSIX definition, and not rely on that specific `fsync()` side effect, as it might change in the future.

#### Using fast commits

Fast commits are activated at filesystem creation time, so users will have to recreate their filesystems to use this feature. In addition, the required [support in `e2fsprogs`](<https://github.com/harshadjs/e2fsprogs/tree/fast-commit-v2>) has not yet been added to the [main branch](<https://git.kernel.org/pub/scm/fs/ext2/e2fsprogs.git>), but is still in development. So interested users will need to compile the tool on their own, or wait until the feature is supported by their distribution. When enabled, information on fast commits shows up in a new `/proc/fs/ext4/_dev_ /fc_info` file.

On the development side, there are numerous features to be added to fast commits. These include making the operations more fine-grained and supporting more cases that fall back to standard commits today. Shirwadkar [is also working](<https://github.com/harshadjs/linux/tree/fc-pmem-renewed>) on fast commits with byte-granularity (instead of the current block-granularity) support for direct-access ([DAX](<https://www.kernel.org/doc/Documentation/filesystems/dax.txt>)) mode, to be used on [persistent memory](<https://en.wikipedia.org/wiki/Persistent_memory>) devices.

The benchmark results given by Shirwadkar in the posted patch set show 20-200% performance improvements with filesystem benchmarks for local filesystems, and 30-75% improvement for NFS workloads. We can assume that the performance gain will be more important in applications doing many `fsync()` operations than in those doing only a few. Either way, though, the fast-commits feature should lead to better ext4 filesystem performance going forward.
