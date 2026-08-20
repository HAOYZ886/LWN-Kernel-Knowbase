---
title: The second half of the 3.2 merge window
url: https://lwn.net/Articles/465738/
date: "November 8, 2011"
category: "Releases-3.2"
author: "By Jonathan Corbet November 8, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 8, 2011

Linus [announced](<https://lwn.net/Articles/466253/>) the 3.2-rc1 release and closed the merge window on November 7\. During the two-week window, some 10,214 non-merge changesets were pulled into the mainline kernel. That is the most active merge window ever, edging past the previous record holder (2.6.30, at 9,603 changesets) by a fair amount. The delay in the start of this development cycle will certainly have caused more work to pile up, but there was also, clearly, just a lot of work going on. 

User-visible changes merged since [last week's summary](<https://lwn.net/Articles/465066/>) include: 

  * The device mapper has a new "thin provisioning" capability which, among other things, offers improved snapshot support. This feature is considered experimental in 3.2. See [Documentation/device-mapper/thin-provisioning.txt](<https://lwn.net/Articles/465740/>) for information on how it works. Also added to the device mapper is a "bufio" module that adds another layer of buffering between the system and a block device; the thin provisioning code is the main user of this feature. 

  * There is a new memory-mapped virtio device intended to allow virtualized guests to use virtio-based block and network devices in the absence of PCI support. 

  * It is now possible for a process to use `poll()` on files under `/proc/sys`; the result is the ability to get a notification when a specific sysctl parameter changes. 

  * The btrfs filesystem now records a number of previous tree roots which can be useful in recovering damaged filesystems; see [this article](<https://lwn.net/Articles/465160/>) for more information. Btrfs has also gained improved readahead support. 

  * The [I/O-less dirty throttling](<https://lwn.net/Articles/456904/>) patch set has been merged; that should improve writeback performance for a number of workloads. 

  * New drivers include: 

    * **Processors and systems** : Freescale P3060 QDS boards and non-virtualized PowerPC systems. 

    * **Block** : M-Systems Disk-On-Chip G3 MTD controllers. 

    * **Media** : MaxLinear MXL111SF DVB-T demodulators, Abilis AS102 DVB receivers, and Samsung S5K6AAFX sensors. 

    * **Miscellaneous** : Intel Sandybridge integrated memory controllers, Intel Medfield MSIC (audio/battery/GPIO...) controllers, IDT Tsi721 PCI Express SRIO (RapidIO) controllers, GPIO-based pulse-per-second clients, and STE hardware semaphores. 

    * **Graduations** : the Conexant cx25821 V4L2 driver has moved from staging into the mainline.  

Changes visible to kernel developers include: 

  * The new `GENHD_FL_NO_PART_SCAN` device flag suppresses the normal partition scan when a new block device is added to the system. 

  * The venerable block layer function `__make_request()` has been renamed to `blk_queue_bio()` and exported to modules. 

  * The `TAINT_OOT_MODULE` taint flag is now set when out-of-tree modules are inserted into the kernel. Naturally, the module itself tells the kernel about its provenance, so this mechanism can be circumvented, but anybody trying to do that would certainly be caught and publicly shamed sooner or later. 

  * A few macros (`EXPORT_SYMBOL_`* and `THIS_MODULE`) have been split out of `<linux/module.h>` and placed in `<linux/export.h>`. Code that only needs to export symbols can now use the latter include file; the result is a reduction in kernel compile time. 

Despite the size of this development cycle, a number of trees ended up not being pulled. Linus explicitly avoided those that were controversial ([FrontSwap](<https://lwn.net/Articles/465317/>) and the KVM tool, for example); others seem to have simply been passed over. Some may slip in for -rc2, but, for the most part, the time has come to stabilize all of this code. If the usual pattern holds, the 3.2 release can be expected sometime around mid-January.
