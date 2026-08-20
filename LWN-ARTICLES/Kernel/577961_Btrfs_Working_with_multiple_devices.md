---
title: "Btrfs: Working with multiple devices"
url: https://lwn.net/Articles/577961/
date: "December 30, 2013"
category: "Btrfs-LWNs guide to; Filesystems-Btrfs"
author: "By Jonathan Corbet December 30, 2013 LWN's guide to Btrfs"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
December 30, 2013

* * *

[LWN's guide to Btrfs](<https://lwn.net/Articles/576276/>)

The previous installments of this series on the Btrfs filesystem have focused on the basics of using Btrfs like any other Linux filesystem. But Btrfs offers a number of features not supported by the alternatives; near the top of that list is support for multiple physical devices. Btrfs is not just a filesystem; it also has its own [RAID](<https://en.wikipedia.org/wiki/RAID>) mechanism built in. This article will delve into how this feature works and how to make use of it. 

There are two fundamental reasons to want to spread a single filesystem across multiple physical devices: increased capacity and greater reliability. In some configurations, RAID can also offer improved throughput for certain types of workloads, though throughput tends to be a secondary consideration. RAID arrays can be arranged into a number of configurations ("levels") that offer varying trade-offs between these parameters. Btrfs does not support all of the available RAID levels, but it does have support for the levels that most people actually want to use. 

RAID 0 ("striping") can be thought of as a way of concatenating multiple physical disks together into a single, larger virtual drive. A strict striping implementation distributes data across the drives in a well-defined set of "stripes"; as a result, all of the drives must be the same size, and the total capacity is simply the product of the number of drives and the capacity of any individual drive in the array. Btrfs can be a bit more flexible than this, though, supporting a concatenation mode (called "single") which can work with unequally sized drives. In theory, any number of drives can be combined into a RAID 0 or "single" array. 

RAID 1 ("mirroring") trades off capacity for reliability; in a RAID 1 array, two drives (of the same size) store identical copies of all data. The failure of a single drive can kill an entire RAID 0 array, but a RAID 1 array will lose no data in that situation. RAID 1 arrays will be slower for write-heavy use, since all data must be written twice, but they can be faster for read-heavy workloads, since any given read can be satisfied by either drive in the array. 

RAID 10 is a simple combination of RAID 0 and RAID 1; at least two pairs of drives are organized into independent RAID 1 mirrored arrays, then data is striped across those pairs. 

RAID 2, RAID 3, and RAID 4 are not heavily used, and they are not supported by Btrfs. RAID 5 can be thought of as a collection of striped drives with a parity drive added on (in reality, the parity data is usually distributed across all drives). A RAID 5 array with _N_ drives has the storage capacity of a striped array with _N-1_ drives, but it can also survive the failure of any single drive in the array. RAID 6 uses a second parity drive, increasing the amount of space lost to parity blocks but adding the ability to lose two drives simultaneously without losing any data. A RAID 5 array must have at least three drives to make sense, while RAID 6 needs four drives. Both RAID 5 and RAID 6 are supported by Btrfs. 

One other noteworthy point is that Btrfs goes out of its way to treat metadata differently than file data. A loss of metadata can threaten the entire filesystem, while the loss of file data affects only that one file — a lower-cost, if still highly undesirable, failure. Metadata is usually stored in duplicate form in Btrfs filesystems, even when a single drive is in use. But the administrator can explicitly configure how data and metadata are stored on any given array, and the two can be configured differently: data might be simply striped in a RAID 0 configuration, for example, while metadata is stored in RAID 5 mode in the same filesystem. And, for added fun, these parameters can be changed on the fly. 

#### A striping example

Earlier in this series, we used `mkfs.btrfs` to create a simple Btrfs filesystem. A more complete version of this command for the creation of multiple-device arrays looks like this: 

```
mkfs.btrfs -d _mode_ -m _mode_ _dev1_ _dev2_ ...
```

This command will group the given devices together into a single array and build a filesystem on that array. The `-d` option describes how data will be stored on that array; it can be `single`, `raid0`, `raid1`, `raid10`, `raid5`, or `raid6`. The placement of metadata, instead, is controlled with `-m`; in addition to the modes available for `-d`, it supports `dup` (metadata is stored twice somewhere in the filesystem). The storage modes for data and metadata are not required to be the same. 

So, for example, a simple striped array with two drives could be created with: 

```
mkfs.btrfs -d raid0 /dev/sdb1 /dev/sdc1
```

Here, we have specified striping for the data; the default for metadata will be `dup`. This filesystem is mounted with the `mount` command as usual. Either `/dev/sdb1` or `/dev/sdc1` can be specified as the drive containing the filesystem; Btrfs will find all other drives in the array automatically. 

The `df` command will only list the first drive in the array. So, for example, a two-drive RAID 0 filesystem with a bit of data on it looks like this: 

```
# df -h /mnt
        Filesystem      Size  Used Avail Use% Mounted on
        /dev/sdb1       274G   30G  241G  11% /mnt
```

More information can be had with the `btrfs` command: 

```
root@dt:~# btrfs filesystem show /mnt
        Label: none  uuid: 4714fca3-bfcb-4130-ad2f-f560f2e12f8e
    	    Total devices 2 FS bytes used 27.75GiB
    	    devid    1 size 136.72GiB used 17.03GiB path /dev/sdb1
    	    devid    2 size 136.72GiB used 17.01GiB path /dev/sdc1
```

(Subcommands to `btrfs` can be abbreviated, so one could type "`fi`" instead of "`filesystem`", but full commands will be used here). This output shows the data split evenly across the two physical devices; the total space consumed (17GiB on each device) somewhat exceeds the size of the stored data. That shows a commonly encountered characteristic of Btrfs: the amount of free space shown by a command like `df` is almost certainly not the amount of data that can actually be stored on the drive. Here we are seeing the added cost of duplicated metadata, among other things; as we will see below, the discrepancy between the available space shown by `df` and reality is even greater for some of the other storage modes. 

#### Device addition and removal

Naturally, no matter how large a particular filesystem is when the administrator sets it up, it will prove too small in the long run. That is simply one of the universal truths of system administration. Happily, Btrfs makes it easy to respond to a situation like that; adding another drive (call it "`/dev/sdd1`") to the array described above is a simple matter of: 

```
# btrfs device add /dev/sdd1 /mnt
```

Note that this addition can be done while the filesystem is live — no downtime required. Querying the state of the updated filesystem reveals: 

```
# df -h /mnt
        Filesystem      Size  Used Avail Use% Mounted on
        /dev/sdb1       411G   30G  361G   8% /mnt
    
        # btrfs filesystem show /mnt
        Label: none  uuid: 4714fca3-bfcb-4130-ad2f-f560f2e12f8e
    	    Total devices 3 FS bytes used 27.75GiB
    	    devid    1 size 136.72GiB used 17.03GiB path /dev/sdb1
    	    devid    2 size 136.72GiB used 17.01GiB path /dev/sdc1
    	    devid    3 size 136.72GiB used 0.00 path /dev/sdd1
```

The filesystem has been expanded with the addition of the new space, but there is no space consumed on the new drive. It is, thus, not a truly striped filesystem at this point, though the difference can be hard to tell. New data copied into the filesystem will be striped across all three drives, so the amount of used space will remain unbalanced unless explicit action is taken. To balance out the filesystem, run: 

```
# btrfs balance start -d -m /mnt
        Done, had to relocate 23 out of 23 chunks
```

The flags say to balance both data and metadata across the array. A balance operation involves moving a lot of data between drives, so it can take some time to complete; it will also slow access to the filesystem. There are subcommands to pause, resume, and cancel the operation if need be. Once it is complete, the picture of the filesystem looks a little different: 

```
# btrfs filesystem show /mnt
        Label: none  uuid: 4714fca3-bfcb-4130-ad2f-f560f2e12f8e
    	    Total devices 3 FS bytes used 27.78GiB
    	    devid    1 size 136.72GiB used 10.03GiB path /dev/sdb1
    	    devid    2 size 136.72GiB used 10.03GiB path /dev/sdc1
    	    devid    3 size 136.72GiB used 11.00GiB path /dev/sdd1
```

The data has now been balanced (approximately) equally across the three drives in the array. 

Devices can also be removed from an array with a command like: 

```
# btrfs device delete /dev/sdb1 /mnt
```

Before the device can actually removed, it is, of course, necessary to relocate any data stored on that device. So this command, too, can take a long time to run; unlike the `balance` command, `device delete` offers no way to pause and restart the operation. Needless to say, the command will not succeed if there is not sufficient space on the remaining drives to hold the data from the outgoing drive. It will also fail if removing the device would cause the array to fall below the minimum number of drives for the RAID level of the filesystem; a RAID 0 filesystem cannot be left with a single drive, for example. 

Note that any drive can be removed from an array; there is no "primary" drive that must remain. So, for example, a series of `add` and `delete` operations could be used to move a Btrfs filesystem to an entirely new set of physical drives with no downtime. 

#### Other RAID levels

The management of the other RAID levels is similar to RAID 0\. To create a mirrored array, for example, one could run: 

```
mkfs.btrfs -d raid1 -m raid1 /dev/sdb1 /dev/sdc1
```

With this setup, both data and metadata will be mirrored across both drives. Exactly two drives are required for RAID 1 arrays; these arrays, once again, can look a little confusing to tools like `df`: 

```
# du -sh /mnt
        28G	    /mnt
    
        # df -h /mnt
        Filesystem      Size  Used Avail Use% Mounted on
        /dev/sdb1       280G   56G  215G  21% /mnt
```

Here, `df` shows 56GB of space taken, while `du` swears that only half that much data is actually stored there. The listed size of the filesystem is also wrong, in that it shows the total space, not taking into account that every block will be stored twice; a user who attempts to store that much data in the array will be sorely disappointed. Once again, more detailed and correct information can be had with: 

```
# btrfs filesystem show /mnt
        Label: none  uuid: e7e9d7bd-5151-45ab-96c9-e748e2c3ee3b
    	    Total devices 2 FS bytes used 27.76GiB
    	    devid    1 size 136.72GiB used 30.03GiB path /dev/sdb1
    	    devid    2 size 142.31GiB used 30.01GiB path /dev/sdc1
```

Here we see the full data (plus some overhead) stored on each drive. 

A RAID 10 array can be created with the `raid10` profile; this type of array requires an even number of drives, with four drives at a minimum. Drives can be added to — or removed from — an active RAID 10 array, but, again, only in pairs. RAID 5 arrays can be created from any number of drives with a minimum of three; RAID 6 needs a minimum of four drives. These arrays, too, can handle the addition and removal of drives while they are mounted. 

#### Conversion and recovery

Imagine for a moment that a three-device RAID 0 array has been created and populated with a bit of data: 

```
# mkfs.btrfs -d raid0 -m raid0 /dev/sdb1 /dev/sdc1 /dev/sdd1
        # mount /dev/sdb1 /mnt
        # cp -r /random-data /mnt
```

At this point, the state of the array looks somewhat like this: 

```
# btrfs filesystem show /mnt
        Label: none  uuid: 6ca4e92a-566b-486c-a3ce-943700684bea
    	    Total devices 3 FS bytes used 6.57GiB
    	    devid    1 size 136.72GiB used 4.02GiB path /dev/sdb1
    	    devid    2 size 136.72GiB used 4.00GiB path /dev/sdc1
    	    devid    3 size 136.72GiB used 4.00GiB path /dev/sdd1
```

After suffering a routine disk disaster, the system administrator then comes to the conclusion that there is value in redundancy and that, thus, it would be much nicer if the above array used RAID 5 instead. It would be entirely possible to change the setup of this array by backing it up, creating a new filesystem in RAID 5 mode, and restoring the old contents into the new array. But the same task can be accomplished without downtime by converting the array on the fly: 

```
# btrfs balance start -dconvert=raid5 -mconvert=raid5 /mnt
```

(The [balance filters page](<https://btrfs.wiki.kernel.org/index.php/Balance_Filters>) on the Btrfs wiki and [this patch changelog](<http://www.mail-archive.com/linux-btrfs@vger.kernel.org/msg14365.html>) have better information on the `balance` command than the `btrfs` man page). Once again, this operation can take a long time; it involves moving a lot of data between drives and generating checksums for everything. At the end, though, the administrator will have a nicely balanced RAID 5 array without ever having had to take the filesystem offline: 

```
# btrfs filesystem show /mnt
        Label: none  uuid: 6ca4e92a-566b-486c-a3ce-943700684bea
    	    Total devices 3 FS bytes used 9.32GiB
    	    devid    1 size 136.72GiB used 7.06GiB path /dev/sdb1
    	    devid    2 size 136.72GiB used 7.06GiB path /dev/sdc1
    	    devid    3 size 136.72GiB used 7.06GiB path /dev/sdd1
```

Total space consumption has increased, due to the addition of the parity blocks, but otherwise users should not notice the conversion to the RAID 5 organization. 

A redundant configuration does not prevent disk disasters, of course, but it does enable those disasters to be handled with a minimum of pain. Let us imagine that `/dev/sdc1` in the above array starts to show signs of failure. If the administrator has a spare drive (we'll call it `/dev/sde1`) available, it can be swapped into the array with a command like: 

```
btrfs replace start /dev/sdc1 /dev/sde1 /mnt
```

If needed, the `-r` flag will prevent the system from trying to read from the outgoing drive if possible. Replacement operations can be canceled, but they cannot be paused. Once the operation is complete, `/dev/sdc1` will no longer be a part of the array and can be disposed of. 

Should a drive fail outright, it may be necessary to mount the filesystem in the degraded mode (with the "`-o degraded`" flag. The dead drive can then be removed with: 

```
btrfs device delete missing /mnt
```

The word "`missing`" is recognized as meaning a drive that is expected to be part of the array, but which is not actually present. The replacement drive can then be added with `btrfs device add`, probably followed by a balance operation. 

#### Conclusion

The multiple-device features have been part of the Btrfs design from the early days, and, for the most part, this code has been in the mainline and relatively stable for some time. The biggest exception is the RAID 5 and RAID 6 support, which was merged for 3.9. Your editor has not seen huge numbers of problem reports for this functionality, but the fact remains that it is relatively new and there may well be a surprise or two there that users have not yet encountered. 

Built-in support for RAID arrays is one of the key Btrfs features, but the list of advanced capabilities does not stop there. Another fundamental aspect of Btrfs is its support for subvolumes and snapshots; those will be discussed in the next installment in this series.
