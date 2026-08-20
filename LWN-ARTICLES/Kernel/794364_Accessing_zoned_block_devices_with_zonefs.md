---
title: Accessing zoned block devices with zonefs
url: https://lwn.net/Articles/794364/
date: "July 23, 2019"
category: "Block layer-Zoned devices; Filesystems"
author: "By Jake Edge July 23, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
July 23, 2019

Zoned block devices are quite different than the block devices most people are used to. The concept came from [shingled magnetic recording](<https://en.wikipedia.org/wiki/Shingled_magnetic_recording>) (SMR) devices, which allow much higher density storage, but that extra capacity comes with a price: less flexibility. Zoned devices have regions (zones) that can only be written sequentially; there is no random access for writes to those zones. Linux already supports these devices, and filesystems are adding support as well, but some applications may want a simpler, more straightforward interface; that's what a new filesystem, zonefs, is targeting. 

Damien Le Moal [posted](<https://lwn.net/ml/linux-fsdevel/20190712030017.14321-1-damien.lemoal@wdc.com/>) an RFC patch series for zonefs to the linux-fsdevel mailing list in mid-July. He also [spoke about zonefs](<https://lwn.net/Articles/788851/>) at the Linux Storage, Filesystem, and Memory-Management Summit (LSFMM) back in May. It is a way for applications to use the POSIX file API, ""rather than relying on direct block device file ioctls and read/write"". Applications that use [log-structured merge-trees](<https://en.wikipedia.org/wiki/Log-structured_merge-tree>) (such as [RocksDB](<https://rocksdb.org/>) and [LevelDB](<https://github.com/google/leveldb>)) will be able to use zoned block devices more easily via zonefs, Le Moal said. 

Zoned block devices typically have both conventional zones—those that allow normal random-access reads and writes—and sequential zones, which only allow writing to the end of the zone. Sequential zones each have a write pointer stored by the device that indicates where the next write operation will be done for that zone. Zonefs simply exposes the zones as files in its filesystem. 

A mounted zonefs will have two top-level directories: `cnv` for conventional zones and `seq` for sequential zones. Those directories will contain a fixed set of files that correspond to the zones in the device. By default, those files will be named with consecutive integers representing the order of the zones reported by [`blkdev_report_zones()`](<https://elixir.bootlin.com/linux/v5.2.2/source/block/blk-zoned.c#L160>) when the filesystem is mounted; zones will effectively be numbered based on the order of their starting sector. A mounted filesystem might look something like this: 

```
mnt/
        |
        |--- cnv/
        |    |--- 0
        |    |--- 1
        |    |--- 2
        |    ...
        |
        |--- seq/
             |--- 0
             |--- 1
             |--- 2
             |--- 3
             ...
```

The first zone is reserved for a superblock, so it does not appear in the hierarchy. The superblock has just a little bit of metadata: a magic number, a UUID, and some feature flags that were given as part of the filesystem create operation (which is done with `mkzonefs`). One of the feature flags will cause zonefs to aggregate all of the conventional zones into a single zone; conventional zones tend to be much smaller on these devices, so aggregation may well make sense. A normal Linux filesystem could be created on the aggregated zone, for example. The default file-name scheme can also be changed by a feature flag to have the file names reflect the sector number of the start of the zone instead. The other two flags will set the user and group IDs (`root.root` by default) or the file permissions (`0640` by default). 

The filesystem is very restrictive; no files or directories can be created on it, for example, nor can files have their owners or permissions changed. The conventional zones cannot be truncated and the sequential zones can only be truncated to zero, which allows them to be completely overwritten. Any read or write beyond the size of the underlying zone will result in an `EFBIG` ("File too large") error. The reported file size will be the full size of the conventional zone (or zones if they are aggregated); for sequential zones it will be the location of the write pointer. 

Johannes Thumshirn, who contributed some code to zonefs (as did Christoph Hellwig), [wondered](<https://lwn.net/ml/linux-fsdevel/20190712080022.GA16276@x250.microfocus.com/>) if the UID/GID and permissions should be set via mount options, rather than only at filesystem-creation time; a filesystem feature flag could still govern the ability to change those attributes. Le Moal [replied](<https://lwn.net/ml/linux-fsdevel/BN8PR04MB581241A65E81F79882508F4BE7F20@BN8PR04MB5812.namprd04.prod.outlook.com/>) that he had implemented that feature along the way, but decided against keeping it: 

I switched to the static format time definition only so that the resulting operation of the FS is a little more like a normal file system, namely, mounting the device does not change file attributes and so can be mounted and seen with the same attribute no matter where it is mounted, regardless of the mount options. 

Thumshirn [agreed](<https://lwn.net/ml/linux-fsdevel/20190712084718.GB16276@x250.microfocus.com/>) with Le Moal's thinking but has a different use case in mind. SMR drives could be formatted for zonefs, then handed out to various administrators who could determine the right UID/GID and permissions for their application. This is an area that requires some more thinking, Thumshirn said. 

Jeff Moyer [expressed concern](<https://lwn.net/ml/linux-fsdevel/x49h87iqexz.fsf@segfault.boston.devel.redhat.com/>) that zonefs breaks most of the expectations that users have for what a filesystem is. He would rather see some other solution, [such as a user-space library](<https://lwn.net/ml/linux-fsdevel/x49zhlbe8li.fsf@segfault.boston.devel.redhat.com/>) (which Le Moal [said](<https://lwn.net/ml/linux-fsdevel/BYAPR04MB5816B59932372E2D97330308E7C80@BYAPR04MB5816.namprd04.prod.outlook.com/>) he had considered) or perhaps a device-mapper target that exposed each zone as a separate block device. Le Moal [pointed out](<https://lwn.net/ml/linux-fsdevel/BYAPR04MB5816A2630B1EBC0696CBEC71E7CA0@BYAPR04MB5816.namprd04.prod.outlook.com/>) that handling each zone as a block device is problematic: 

Well, I do not think you need a new device mapper for this. dm-linear supports zoned block devices and will happily allow mapping a single zone and expose a block device file for it. My problem with this approach is that SMR drives are huge, and getting bigger. A 15 TB drive has 55380 zones of 256 MB. Upcoming 20 TB drives have more than 75000 zones. Using dm-linear or any per-zone device mapper target would create a huge resources pressure as the amount of memory alone that would be used per zone would be much higher than with a file system and the setup would also take far longer to complete compared to zonefs mount. 

Dave Chinner [agreed](<https://lwn.net/ml/linux-fsdevel/20190722001200.GQ7689@dread.disaster.area/>) with that assessment. Le Moal said that he would rather point people at a regular filesystem that has zoned block device support, such as Btrfs, where the feature is [in progress](<https://lwn.net/Articles/790652/>), or, eventually, XFS (which is planned), but that some application developers often want to dispense with most or all of what filesystems provide. The idea is that zonefs provides just enough of a filesystem for those developers: ""zonefs fits in the middle ground here between removing the normal file system and going to raw block device"". 

No strong objections were heard in the thread (or in the LSFMM session, for that matter). It is a bit of a strange filesystem, but would provide easy access to these zoned block devices from applications. The semantics of a "file" (especially in the `seq` directory) would be rather different than the usual POSIX semantics, but would be precisely what certain applications need. The next step would seemingly be to bring zonefs to the Linux kernel mailing list and from there, perhaps, into the mainline in a cycle or two.
