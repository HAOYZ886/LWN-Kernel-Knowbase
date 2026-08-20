---
title: Zoned storage
url: https://lwn.net/Articles/897263/
date: "June 14, 2022"
category: "Block layer-Zoned devices; Filesystems"
author: "By Jake Edge June 14, 2022 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
June 14, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

[Zoned storage](<https://zonedstorage.io/>) is a form of storage that offers higher capacities by making tradeoffs in the kinds of writes that are allowed to the device. It was the topic of a storage and filesystem session led by Luis Chamberlain at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). Over the years, zoned storage has been a frequent topic at LSFMM, [going back to LSFMM 2013](<https://lwn.net/Articles/548116/>), where support for shingled magnetic recording (SMR) devices, which were the starting point for zoned storage, was discussed. 

Chamberlain began with the news that a [zoned storage microconference](<https://lpc.events/event/16/contributions/1147/>) had been accepted for this year's [Linux Plumbers Conference](<https://lpc.events/>) (LPC). He encouraged attendees to submit topics and hoped it was an opportunity to introduce more user-space developers to zoned-storage concepts. LPC will be held September 12-14 in Dublin, Ireland. 

[ ![\[Luis Chamberlain\]](https://static.lwn.net/images/2022/lsfmm-chamberlain2-sm.png) ](<https://lwn.net/Articles/897781/>)

In a "really fast intro" to zoned storage, he quoted from the zoned-storage home page linked above: it is ""a class of storage devices that enables host and storage devices to cooperate to achieve higher storage capacities, increased throughput, and lower latencies"". It comes in different form factors, including SMR and "the latest and trendiest one", which is based on SSDs using [NVMe zoned namespaces](<https://zonedstorage.io/docs/introduction/zns>) (ZNS). The storage device is divided into zones; "sequential zones" are those where all writes in a zone must be sequential and the only way to overwrite the zone is to reset its write pointer to the beginning. Both SMR and ZNS devices can optionally also have "conventional zones" (or namespaces) that support random writes. There can be drives that only have sequential zones, however, which is something that filesystem developers need to keep in mind, he said. 

There are some "ecosystem considerations" to also keep in mind. For one, replacing a zoned-storage device can only be done with another device that has the same zone sizes. Also, ZNS requires manually switching the kernel I/O scheduler to mq-deadline. That is only true for some filesystems, though, an audience member said. Damien Le Moal said that the underlying issue is that the order of writes needs to be guaranteed and the scheduler was the easiest place to ensure that. 

Ted Ts'o said there is a philosophical question on whether the kernel should be tracking the write pointers for the zones or whether user space should be doing that. At Google, there is an out-of-tree patch set that hacks the CFQ scheduler so that it will not merge requests and gives user space the responsibility to track write pointer, but he knows that this out-of-tree approach is not sustainable long term. It was generally agreed that there is much to discuss around these topics at LPC. 

Moving on, Chamberlain noted that the patches for supporting zone sizes that are not powers of two (npo2) had been [posted](<https://lwn.net/ml/linux-kernel/20220427160255.300418-1-p.raghav%40samsung.com/>) and that more revisions should be coming soon (v6 was [posted](<https://lwn.net/ml/linux-kernel/20220525154957.393656-1-p.raghav%40samsung.com/>) on May 25). He also said that `btrfs check` currently does not work for image dumps of any zoned storage, probably due to a lack of zone information on the disk. There are also some questions about how to write the superblocks for Btrfs on not-power-of-2 devices. 

Hearkening back to an [earlier session](<https://lwn.net/Articles/897202/>), Chamberlain said that the lack of support for `ioctl()` and direct I/O (i.e. `O_DIRECT`) in Java causes problems for using it with [zonefs](<https://lwn.net/Articles/794364/>), which requires direct I/O. Le Moal said that the long-term goal is to remove the direct I/O requirement, but it is currently needed to maintain the write-order guarantees. 

After Chamberlain asked about the status of [bcachefs](<https://lwn.net/Articles/895266/>), Kent Overstreet said that it already has native zoned-device support. The allocation mechanism for bcachefs has always been bucket-based, [going back to bcache](<https://www.kernel.org/doc/html/v5.18/admin-guide/bcache.html>), which is a good match to zoned storage. Those devices will work just as well as regular block devices for bcachefs, he said. 

Chamberlain said he has been using [kdevops](<https://github.com/mcgrof/kdevops>) to "test the hell out of zoned-storage devices" with both fstests and blktests. There are two modes of testing that can be done, either with real hardware or using QEMU virtual devices. There are some problems, currently, with the QEMU driver for ZNS devices, Le Moal said. 

Bart Van Assche wondered if there was a need to keep an eye on the standards committees because it seems like there is a gap between the feature sets offered by SCSI and NVMe. Today the zoned-device code is shared for SCSI and NVMe in the kernel, but he worries that may have to change. In particular, filesystems might be forced to choose which of the two types they support. Le Moal said that there are features in ZNS that are not available for SCSI; Btrfs has been trying to adapt to those differences, for example. 

Van Assche is working on Android devices with zoned storage. The devices are currently SCSI, but will eventually be NVMe-based; he is concerned that the filesystem being used will have to change because of that. Le Moal said that the goal is that the filesystems should not have to care about the type of the underlying device. Several others said that because the underlying storage technology is quite different, though, the kernel may end up with two APIs for zoned storage. To a certain extent, the differences in the I/O scheduler reflect that, one attendee said. 

Josef Bacik said that there should be only minimal differences that filesystems need to be aware of in order to support zoned storage. There are already changes in Btrfs to support the idea of zones, but that is about as far as things should go; anything further should be pushed out to user space, he said. Ts'o said that the block layer is the right place to handle these extra features and to do so in a way that hides the underlying differences from filesystems. He noted that features like [discard](<https://lwn.net/Kernel/Index/#Block_layer-Discard_operations>) are used by filesystems through the block-layer interface, which hides the device differences. 

If there are new features that can provide a large benefit, storage-device developers should have a conversation with the filesystem community about them, Ts'o said. If, for example, providing a hint that some data is for a journal, such as for a filesystem or database, would make a big performance difference, there may be a way to change the filesystems and applications to take advantage of it. But the benefit needs to be large and the interface to the feature needs to be stable. "It would probably be an `ioctl()`", he said to chuckles, but he could imagine adding a feature of that sort.
