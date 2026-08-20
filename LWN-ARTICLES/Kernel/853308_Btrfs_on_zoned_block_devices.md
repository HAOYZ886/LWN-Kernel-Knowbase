---
title: Btrfs on zoned block devices
url: https://lwn.net/Articles/853308/
date: "April 19, 2021"
category: "Block layer-Zoned devices; Filesystems-Btrfs"
author: "April 19, 2021 This article was contributed by Marta Rybczyńska"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

April 19, 2021

This article was contributed by Marta Rybczyńska

[Zoned block devices](<https://zonedstorage.io/introduction/zoned-storage/>) have some unfamiliar characteristics that result from compromises made in the name of higher storage density. They are divided into zones, some or all of which do not support random access for write operations. Instead, these "sequential" zones can only be written in order, from the first block to the last. This constraint poses a new challenge for filesystems, which are normally designed with the assumption that storage blocks can be written in any order. It is thus not surprising that zoned-device support in mainstream filesystems in Linux has been slow in coming; that is changing, though, with the [addition](<https://lwn.net/ml/linux-fsdevel/cover.1612433345.git.naohiro.aota@wdc.com/>) of support for zoned block devices to Btrfs in Linux 5.12.

The only way to overwrite data in a zoned drive's sequential zone is to reset the write pointer to the beginning of the zone, which immediately erases the entire content of that zone. On the other hand, random read access is fully supported. Many zoned devices also provide some "conventional" zones that support random read and write operations. Zoned devices were first seen in the form of [shingled magnetic recording](<https://en.wikipedia.org/wiki/Shingled_magnetic_recording>) (SMR) drives; the kernel has low-level support for these devices. Zoned devices using flash storage also exist; they trade flexibility for reduced hardware complexity. These devices were added to the NVMe standard in the form of [the Zoned Namespaces (ZNS) command set](<https://zonedstorage.io/introduction/zns/>), which has been supported in Linux since the 5.9 release. 

Work has been going on for a number of years to support zoned drives in Linux filesystems. Copy-on-write filesystems should be easier to adapt, as they are designed to avoid overwriting data blocks. Among the existing Linux filesystems, [F2FS](<https://en.wikipedia.org/wiki/F2FS>) already supports zoned devices, and allows normal operations on such devices (but requires that the drive provide at least one conventional zone). In addition, zonefs, a special filesystem for zoned devices, [was included](<https://lwn.net/Articles/794364/>) in the 5.6 kernel. Using zonefs requires applications designed for this purpose, as the filesystem does not support the creation of normal files. Some types of applications do fit the model well, however, for example those with log-structured data.

Support for zoned devices in more mainstream filesystems has also been in progress in recent years. This is the case for Btrfs, which has seen the support of zoned devices in the works since at least 2019, when Noahiro Aota [presented the status](<https://lwn.net/Articles/788851/>) of this work at the 2019 Linux Storage, Filesystem, and Memory-Management Summit. He has continued the work since, and finally has seen the 15th revision merged into 5.12.

#### Changing Btrfs

Supporting zoned devices requires changes in the way the filesystem structures are organized on disk, as it is often impossible to overwrite existing data. One implication is that data structures that change over time must either be placed in a conventional zone or be implemented in a way that does not require them to be in a single, fixed location. 

The only on-disk structure that had a fixed location in Btrfs was the superblock, which may have [up to two copies](<https://btrfs.wiki.kernel.org/index.php/On-disk_Format#Superblock>). In the case of zoned devices, an easy solution would be to require the availability of conventional zones for the superblock and its copies. However, the location of the second copy [falls into a sequential zone](<https://lwn.net/ml/linux-btrfs/5aa30b45e2e29018e19e47181586f3f436759b69.1605007036.git.naohiro.aota@wdc.com/>) on many existing devices; in addition, devices with no conventional zones should also be supported. For those reasons, Aota implemented a log-based superblock using two zones as circular buffers to [protect against power failures](<https://lwn.net/ml/linux-fsdevel/SN4PR0401MB35987EE941FA59E2ECB8D7269B8C9@SN4PR0401MB3598.namprd04.prod.outlook.com/>). The filesystem writes the updated superblocks sequentially to the first zone, and switches to the second one when the first one is full. It can then reset the first zone safely.

Another structure that required changes was the tree log. Blocks for this log were allocated together with other types of metadata blocks; since the tree log is written at a different time than other metadata, this approach would generate non-sequential writes. The [solution](<https://lwn.net/ml/linux-fsdevel/4a02c3ff283a1c2d71bfa3b0a7135b062af7385e.1612434091.git.naohiro.aota@wdc.com/>) is to separate the tree-log blocks from the other metadata; then each data stream can be written separately and sequentially.

At the lower level, other changes impact Btrfs chunks (also called block groups), which are a Btrfs data structure that represents a range of data blocks on disk. In the case of zoned Btrfs, the default chunk size was changed so that it is aligned to the zone size. The allocation of blocks in a chunk also changes to meet the zoned-device requirements: it becomes sequential from the beginning of the chunk. If blocks are freed behind the allocation pointer, they will be ignored by further allocations. As a result, blocks will be always allocated sequentially. The zone of a specific chunk will be allowed for reuse (reset) only when all blocks in the chunk are freed.

In addition to the data-layout modifications, Btrfs will write to sequential zones using the "zone append" command (represented by the `REQ_OP_ZONE_APPEND` block-device operation) on the underlying device instead of using a simple write. This command needs a zone to operate on, but does not require a write pointer; it returns address of the written block. This means that the order in which structures are written does not matter and a block-group lock is not required; the performance improvement reported by Aota in the cover letter is 36% for 4KB random writes.

With the patch set, Btrfs supports NVMe drives with the ZNS functionality natively, and the `sd` driver provides emulation of the zoned command set for SAS and SATA hard drives.

#### How it works in 5.12

The current zoned-device support shows a number of limitations and areas needing further improvements. The high-level list appears in the [merge request for 5.12](<https://lwn.net/ml/linux-btrfs/cover.1613417746.git.dsterba%40suse.com/>), and the complete one in the cover letter.

The first issue that users may face is that 5.12 supports only the _single_ Btrfs [profile](<https://btrfs.wiki.kernel.org/index.php/Manpage/mkfs.btrfs#PROFILES>). That means no support for features like data duplication and RAID. The difficulty of supporting other profiles lies in the need to handle separate logical addresses for the same data in different zones. The example Aota gives in the cover letter is the DUP profile (duplicating data). In this case, when the filesystem issues an append command, two zones may respond with different offsets.

Administrators should be also aware that if a volume contains multiple zoned disks, the zone sizes for all disks should be the same.

The current [`btrfs-progs`](<https://github.com/kdave/btrfs-progs>) release does not yet support zoned devices; a branch with that support can be found in [Aota's repository](<https://github.com/naota/btrfs-progs/tree/btrfs-zoned>). 

I tested a zoned Btrfs filesystem alongside a non-zoned one, using same size partitions on a SATA drive. The zoned device was emulated. To create a zoned Btrfs filesystem, one needs to use Aota's version of `mkfs.btrfs` with the `-O zoned` option, but also force data and metadata into the "single" profile:

```
mkfs.btrfs /dev/sdb1 -O zoned -d single -m single
```

The two filesystems worked the same way for the most part. The performance was also roughly the same. Btrfs features like creating snapshots also worked equally well on zoned and non-zoned devices. The only difference one could notice is higher space usage on the zoned filesystem, as would be expected. 

#### Conclusions

Zoned block devices behave differently than the traditional ones, so there is no surprise that a filesystem requires layout changes to support those devices, and that the development took many months. For Btrfs, the basic support is present now, so interested users may test it. This support has limitations, however, and we can expect that additional features will be added in upcoming kernel versions.
