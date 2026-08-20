---
title: What got into 2.6.25
url: https://lwn.net/Articles/266721/
date: "January 30, 2008"
category: "Releases-2.6.25"
author: "By Jonathan Corbet January 30, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 30, 2008

As of this writing, some 3800 patches have been merged into the mainline git repository since the release of 2.6.24. That is fewer than one might have expected, but Linus's travel to linux.conf.au is slowing the process somewhat. Expect more than the usual amount of interesting stuff to be merged relatively late in the merge window period. 

User-visible changes include: 

  * New drivers have been added for Globe Trotter HSDPA wireless cards, HIFN 795x crypto accelerator chips, Xceive xc2028 and xc5000 tuners, Cirrus Logic CS5345 analog-to-digital converters, several Beholder TV tuners, Syntek DC1125 cameras, Silicon Labs Si470x FM radio receivers, Atmel AT91CAP9 processors, Qualcomm MSM7X00A processors, Marvell Orion system-on-a-chip devices, Marvell Feroceon processors, SuperH 7203 and 7263 processors, SGI IP28 systems, R6040 Ethernet adapters, Broadcom NetXtremeII 10Gb network adapters, RTL8180 and 8185-based wireless network cards, Microchip EN28J60 Ethernet chips, and, finally, Atheros-based wireless network adapters. 

  * The Seagate ST-02/Future Domain TMC-8xx and PSI240i SCSI drivers have been removed due to lack of interest and maintenance. 

  * Salsa20 stream cipher support has been added to the crypto layer (at least for the x86 architecture - it's an assembly implementation). 

  * Some realtime work has gone into the scheduler; in particular, the kernel will be more aggressive about moving tasks between processors when multiple realtime tasks are contending for the same CPU. The implementation of cpusets has been made to work more with the scheduler domains mechanism. The option to make the big kernel lock preemptible has been made the default; eventually the non-preemptible version will go away altogether. High-resolution timers can be used for preemption, making fair scheduling more accurate. The group scheduling feature has been enhanced with realtime support. 

  * The [Preemptible read-copy-update patches](<http://lwn.net/Articles/253651/>) have been merged. 

  * Support for the [LatencyTop utility](<http://lwn.net/Articles/266153/>) has been merged. 

  * Kprobes support for the ARM architecture has been added. 

  * The new `CLONE_IO` flag to `clone()` causes I/O contexts (used in the CFQ block I/O scheduler) to be shared with the new child process. 

  * The idle class for I/O scheduling has been changed to not be 100% idle when the device is busy; as a result, it is far less likely to cause priority inversion problems and is no longer limited to privileged processes. 

  * A [long list of new ext4 features](<http://lwn.net/Articles/266274/>), including large file support, (very) large filesystem support, journal checksumming, multi-block allocation, and more, has been added in. 

  * The `splice()` system call now supports TCP receive streams. 

  * [Controller area network](<http://lwn.net/Articles/253425/>) protocol support has been merged. 

  * The network traffic shaper, long obsolete and scheduled for removal, is gone. 

  * Quite a bit of work has been done on the network namespace code which was first merged in 2.6.24. Extending namespace awareness through the entire networking subsystem is a big job which is, at this point, mostly complete. 

Changes visible to kernel developers include: 

  * Chinese translations of a number of core kernel development documents have been added to the tree. 

  * There have been a great many changes to the low-level device model APIs dealing with kobjects and ksets. These changes have, in turn, forced a large number of adjustments throughout the tree. See [Documentation/kobject.txt](<https://lwn.net/Articles/266722/>) for an overview of the new API. 

  * There is a new set of security module functions for dealing with filesystem mount and unmount operations. 

  * The chained scatterlist API has been augmented with the [sg_table](<http://lwn.net/Articles/263343/>) patches. 

  * There have been some changes to the block request completion API. See [this article](<http://lwn.net/Articles/266914/>) for a description of the new way of doing things. 

As of this writing, the merging process has just begun, so expect a long list again next week. Among other things, the [x86 tree update](<http://lwn.net/Articles/267135/>), with 908 changesets, is waiting on the wings. There is quite a bit of code yet to be merged for this development cycle.
