---
title: CDROM drives and partitioning
url: https://lwn.net/Articles/72929/
date: "February 25, 2004"
category: Block layer
author: 
---

It is rare for a CD to be built with partitions; in the modern world, a CD's capacity is considered small enough as it is without splitting it up further. Many of the other reasons for using partitions (robustness in case one partition's filesystem gets corrupted, containing excessive space usage, etc.) also do not apply to the CD medium. As a result, the Linux CD driver does not support partitioning at all. 

It turns out, however, that some companies do produce CDs with partition tables on them. Linux systems will be unable to mount and read the filesystems on such CDs. Most users have never encountered this problem, but, for those who have, Steven Hill has posted [a patch](<https://lwn.net/Articles/72676/>) which adds CDROM partition support to the SCSI CDROM driver. 

The good news is that, in the 2.6 kernel, the block layer handles partitioning. So the active part of the patch boils down to the following: 

```
-	disk = alloc_disk(1);
        +	disk = alloc_disk(partitions + 1);
```

So it turns out to be a relatively easy patch to design and implement. (See [this Driver Porting Series article](<https://lwn.net/Articles/25711/>) for details `alloc_disk()` and the rest of the 2.6 gendisk interface). 

The only problem is that, as one might expect, the minor device numbers for the partitions will be allocated immediately after the minor number for the CD device as a whole. `/dev/scd0`, the first SCSI CDROM device, has device number 11,0, so the first partition on that device would be assigned numbers 11,1. The only problem is that 11,1 is where most systems expect to find `/dev/scd1`, the second CDROM device. No space was ever set aside for partitions in the SCSI CDROM device number range. 

In the relatively near future, dealing with this sort of issue will not be a problem; a small set of `udev` rules will ensure that the right device names are created to correspond to the hardware which is actually present on the system. Until then, however, users of partitioned CDs will have to deal with a conflict in how the kernel and the distributions see the SCSI CD device number space.
