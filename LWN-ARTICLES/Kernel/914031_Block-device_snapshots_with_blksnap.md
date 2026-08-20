---
title: "Block-device snapshots with blksnap"
url: https://lwn.net/Articles/914031/
date: "November 14, 2022"
category: Block layer
author: "By Jonathan Corbet November 14, 2022"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 14, 2022

As a general rule, one need not have worked in the technology industry for long before the value of good data backups becomes clear. Creating a backup that is truly good, though, can be a challenge if the filesystem in question is actively being changed while the backup process runs. Over the years, various ways of addressing this problem have been developed, ranging from simply shutting down the system while backups run to a variety of snapshotting mechanisms. The kernel may be about to get another approach to snapshots should [the blksnap patch set](<https://lwn.net/ml/linux-kernel/20221102155101.4550-1-sergei.shtepa@veeam.com/>) from Sergei Shtepa find its way into the mainline. 

The blksnap patches are rigorously undocumented, so much of what follows comes from reverse-engineering the code. Blksnap performs snapshotting at the block-device level, meaning that it is entirely transparent to any filesystems that may be stored on the devices in question. It is able to create snapshots of a set of multiple block devices, so it should be suitable for RAID arrays and such. The targeted use case appears to be automated backup systems; the snapshots that blksnap creates are described as "non-persistent" and are meant to be discarded once a real backup has been made. 

Since blksnap works at the block level, it must be given space to store snapshots that is separate from the devices being snapshotted. Specifically, there are `ioctl()` operations to assign ranges of sectors on a separate device for the storage of "difference blocks" and to change those assignments over time. There is a notification mechanism whereby a user-space process can be told when a given difference area is running low on space so that it can assign more blocks to that area. 

The algorithm used by blksnap is simple enough: once a snapshot has been created for a set of block devices (using another `ioctl()` operation), blksnap will intercept every block-write operation to those devices. If a given block is being written to for the first time after the snapshot was taken, the previous contents of that block will be copied to the difference area, and a note will be made that the block has been changed since the snapshot was created. Once that is done, the write operation can continue normally. The block devices thus always reflect the most recent writes, while the difference area contains the older data needed to recreate the state of those devices at the time the snapshot was created. 

In order to be able to intercept writes to the block devices, Shtepa has had to add [a new "device filter" mechanism](<https://lwn.net/ml/linux-kernel/20221102155101.4550-2-sergei.shtepa@veeam.com/>) to the block layer. A filter can be attached to a specific device that will be called prior to the execution of each operation on that device, with the BIO structure representing that operation as a parameter. If the filter function returns false, the operation will not be executed. An [earlier version of the patch set](<https://lwn.net/ml/linux-kernel/1655135593-1900-1-git-send-email-sergei.shtepa@veeam.com/>) provided the ability to attach multiple filters to a block device at different "altitudes", but that was removed since there are no other uses for filters currently. 

Blksnap uses the filter function to catch writes to the snapshotted device(s). When a write is found, the operation is put on hold while the original contents of the blocks to be written are copied to the difference area; once that is complete, the write is submitted normally. 

Interestingly, nothing in the patch set describes how one might gain access to a snapshot once it has been created. A look at [the `ioctl()` interface](<https://lwn.net/ml/linux-kernel/20221102155101.4550-3-sergei.shtepa@veeam.com/>) shows a couple of possibilities, though. One is an operation to obtain the list of changed blocks associated with a snapshot, which might be useful for certain types of incremental backups. But blksnap also creates a new, read-only device for each snapshot taken. Reading a block from that device causes blksnap to consult its map of changed blocks; if the block in question has been changed, it is read from the difference area. Otherwise, it can be read from the original block device. The major and minor numbers of the snapshot devices can be obtained with another `ioctl()` operation; there is also an undocumented sysfs file that apparently can be consulted. 

The kernel does not lack for the ability to make snapshots now, so one might logically wonder why blksnap is needed. It clearly differs from the snapshot feature offered by filesystems like Btrfs, since blksnap operates at the block-device level. Among other things, blktrace can be used with filesystems that do not, themselves, have a snapshot feature. Btrfs snapshots are stored on the same block device as the filesystem itself, meaning that the two can compete for space, and the space used by snapshots could prevent the writing of data to the live filesystem. Since blksnap stores its snapshot data on a separate device, that data won't get in the way of ongoing operations. If the difference area runs out of space the snapshot will be corrupted, but the device being snapshotted will be unaffected. 

An existing alternative at the block level is the [device mapper snapshot target](<https://docs.kernel.org/next/admin-guide/device-mapper/snapshot.html>). The functionality provided by blksnap is, in many ways, similar to the device mapper; both work by intercepting writes and copying the old data to a separate device. Blksnap can be used without needing to set up the device mapper for the devices to be snapshotted, though. It also claims to have more flexible management of its difference area, especially when multiple devices are being snapshotted together. 

These differences appear to be interesting enough that nobody has, so far, questioned whether blksnap is a useful addition to the kernel. The patch set (despite being marked "v1") is on its second revision, having seen a number of fixes from its first posting in July. With luck, the next revision will incorporate some documentation; then perhaps it will be nearing readiness for inclusion into the mainline.
