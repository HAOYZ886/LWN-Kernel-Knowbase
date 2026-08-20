---
title: Partitioned loopback devices
url: https://lwn.net/Articles/110468/
date: "November 10, 2004"
category: "Block layer-Loopback device; Loopback device; Partitions"
author: 
---

The expanded device number type in the 2.6 kernel makes it possible, at the lowest level, to support vast numbers of partitions on every block device in the system. Unfortunately, the Linux block drivers have not caught up with this change. SCSI, in particular, is still limited to 15 partitions per device. There are a few reasons for this lag, but the largest is simple compatibility: there is no easy way to incorporate support for more partitions without breaking the existing device numbering scheme. The block layer assumes that partitions have consecutive minor numbers, so supporting more partitions means increasing the portion of the minor number which is dedicated to the partition number. But changing the interpretation of minor numbers in this way would break existing systems, and that is something the kernel developers are reluctant to do. 

Carl-Daniel Hailfinger has recently posted [an interesting solution to the partition limit](<https://lwn.net/Articles/110426/>): partitioned loopback devices. A loopback device is a kernel-implemented virtual block device which is backed up by something real - usually a disk partition or a file on a disk somewhere. Common uses for loopback devices include mounting regular files as filesystems or the creation of encrypted filesystems (though the device mapper is the preferred means for the latter application in 2.6). Loopback devices do not support partitions in their own right; they simply provide block-level access to the backing store as a single partition. 

Carl-Daniel noticed, however, that adding partition support to loopback devices would be a relatively straightforward thing to do. In 2.6, partition handing is (finally) part of the block layer; all that is really required to support partitions in the loopback driver is to tell the block layer that those partitions exist. So, with a small patch, each loopback device can have up to 127 partitions. The bulk of the patch, in fact, is there to ensure continued compatibility for users of non-partitioned loopback devices. 

This capability is interesting because it is a simple matter of one `losetup` command to create a loopback interface to a real disk drive. Thus, by using loopback devices in this mode, system administrators can get around the partition limits enforced by the real hardware drivers and divide their disks into lots of tiny little pieces. There is some small overhead associated with using the loopback device, but, for users in need of more partitions, it may well be a price worth paying.
