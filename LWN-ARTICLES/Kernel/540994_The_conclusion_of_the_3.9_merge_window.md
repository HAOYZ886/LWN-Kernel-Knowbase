---
title: The conclusion of the 3.9 merge window
url: https://lwn.net/Articles/540994/
date: "March 5, 2013"
category: "Releases-3.9"
author: "By Jonathan Corbet March 5, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 5, 2013

By the time that Linus [released](<https://lwn.net/Articles/540993/>) the 3.9-rc1 kernel prepatch and closed the merge window for this cycle, he had pulled a total of 10,265 non-merge changesets into the mainline repository. That is just over 2,000 changes since [last week's summary](<https://lwn.net/Articles/539892/>). The most significant user-visible changes merged at the end of the merge window include: 

  * The block I/O controller now has full hierarchical control group support. 

  * The NFS code has gained network namespace support, allowing the operation of per-container NFS servers. 

  * The Intel [PowerClamp driver](<https://lwn.net/Articles/528124/>) has been merged; PowerClamp allows the regulation of a CPU's power consumption through the injection of forced idle states. 

  * The device mapper has gained support for a new "dm-cache" target that is able to use a fast drive (like a solid-state device) as a cache in front of slower storage devices. See [Documentation/device-mapper/cache.txt](<https://lwn.net/Articles/540996/>) for details. 

  * RAID 5 and 6 support for the Btrfs filesystem has been merged at last. 

  * Btrfs defragmentation code has gained snapshot awareness, meaning that sharing of data between snapshots will no longer be lost when defragmentation runs. 

  * Architecture support for the Synopsys ARC and ImgTec Meta architectures has been added. 

  * New hardware support includes: 

    * **Systems and processors** : Marvell Armada XP development boards, Ralink MIPS-based system-on-chip processors, Atheros AP136 reference boards, and Google Pixel laptops. 

    * **Block** : IBM RamSam PCIe Flash SSD devices and Broadcom BCM2835 SD/MMC controllers. 

    * **Display** : TI LP8788 backlight controllers. 

    * **Miscellaneous** : Kirkwood 88F6282 and 88F6283 thermal sensors, Marvell Dove thermal sensors, and Nokia "Retu" watchdog devices.  

Changes visible to kernel developers include: 

  * The `menuconfig` configuration tool now has proper "save" and "load" buttons. 

  * The [rework of the IDR API](<https://lwn.net/Articles/536293/>) has been merged, simplifying code that uses IDR to generate unique integer identifiers. Users throughout the kernel tree have been updated to the new API. 

  * The `hlist_for_each_entry()` iterator has lost the unused "`pos`" parameter. 

At this point, the stabilization period for the 3.9 kernel has begun. If the usual pattern holds, the final 3.9 release can be expected sometime around the beginning of May.
