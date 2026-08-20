---
title: The conclusion of the 3.10 merge window
url: https://lwn.net/Articles/550205/
date: "May 12, 2013"
category: "Releases-3.10"
author: "By Jonathan Corbet May 12, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 12, 2013

By the time Linus [announced](<https://lwn.net/Articles/550269/>) the 3.10-rc1 kernel, he had pulled just short of 12,000 non-merge changesets into the mainline kernel. That makes 3.10 the busiest merge window ever, by over 1,000 patches. The list of changes merged since [the previous 3.10 merge window summary](<https://lwn.net/Articles/549477/>) is relatively small, but it includes some significant changes. The most significant of those changes are: 

  * The [bcache](<https://lwn.net/Articles/497024/>) caching layer has been merged. Bcache allows a fast device (like an SSD) to provide fast caching in front of a slower device; it is designed for fast performance given the constraints of contemporary solid-state devices. See [Documentation/bcache.txt](<https://lwn.net/Articles/550207/>) for more information. 

  * The on-disk representation of extents in Btrfs has been changed to make the structure significantly smaller. ""In practice this results in a 30-35% decrease in the size of our extent tree, which means we COW less and can keep more of the extent tree in memory which makes our heavy metadata operations go much faster."" It is an incompatible format change that must be explicitly enabled when the filesystem is created (or after the fact with `btrfstune`). 

  * The MIPS architecture has gained basic support for virtualization with KVM. MIPS kernels can also now be built using the new "microMIPS" instruction set, with significant space savings. 

  * New hardware support includes Abilis TB10x processors, Freescale ColdFire 537x processors, Freescale M5373EVB boards, Broadcom BCM6362 processors, Ralink RT2880, RT3883, and MT7620 processors, and Armada 370/XP thermal management controllers. 

Changes visible to kernel developers include: 

  * The block layer has gained basic power management support; it is primarily intended to control which I/O requests can pass through to a device while it is suspending or resuming. To that end, power-management-related requests should be marked with the net `__REQ_PM` flag. 

  * A lot of work has gone into the block layer in preparation for "immutable biovecs," a reimplementation of the low-level structure used to represent ranges of blocks for I/O operations. One of the key advantages here seems to be that it becomes possible to create a new biovec that contains a subrange of an existing biovec, leading to fast and efficient request splitting. The completion of this work will presumably show up in 3.11. 

  * The dedicated thread pool implementation used to implement writeback in the memory management subsystem has been replaced by a workqueue. 

If this development cycle follows the usual pattern, the final 3.10 kernel release can be expected in early July. Between now and then, though, there will certainly be a lot of bugs to fix.
