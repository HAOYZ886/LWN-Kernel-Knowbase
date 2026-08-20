---
title: The first half of the 3.3 merge window
url: https://lwn.net/Articles/474645/
date: "January 11, 2012"
category: "Releases-3.3"
author: "By Jonathan Corbet January 11, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 11, 2012

As of this writing, just over 5,700 non-merge changesets have been pulled into the mainline for the 3.3 development cycle. A fair amount of work remains to be pulled, so it looks like another fairly active cycle, though perhaps not quite up to the level of 3.2. 

Some of the more significant, user-visible changes merged so far include: 

  * The "team" network driver - a lightweight mechanism for bonding multiple interfaces together - has been merged. The [libteam project](<https://github.com/jpirko/libteam>) has the user-space code needed to operate this device. 

  * The network priority control group controller has been added. This controller allows the administrator to specify the priority with which members of each control group have access to the network interfaces available on the system. See [net_prio.txt](<https://lwn.net/Articles/474695/>) from the documentation directory for more information. 

  * Also added is the [TCP buffer size controller](<https://lwn.net/Articles/470656/>) which can be used to place limits on the amount of kernel memory used to hold TCP buffers. 

  * The [byte queue limits](<https://lwn.net/Articles/454390/>) infrastructure has been added, enabling control over how much data can be queued for transmission over a network interface at any time. 

  * The [Open vSwitch](<https://lwn.net/Articles/469775/>) virtual network switch has been merged. 

  * The ARM architecture has gained support for the "large physical address extension," allowing 32-bit processors to address more than 4GB of installed memory. 

  * The "adaptive RED" queue management algorithm is now supported by the networking layer. 

  * The near-field communications (NFC) layer has gained support for the logical link control protocol (LLCP). 

  * The beginnings of [dynamic frequency selection](<https://lwn.net/Articles/420623/>) support have been added to the wireless networking subsystem. 

  * For S390 users who find the current limit of 3.8TB of RAM to be constraining: 3.3 will add support for four-level page tables and an upper limit of 64TB (for now). 

  * Various Android drivers have returned to the staging tree; see [this article](<https://lwn.net/Articles/472984/>) for more information. 

  * The C6X architecture (described in [this article](<https://lwn.net/Articles/457635/>)) has been merged. 

  * The ext4 filesystem has added support for online resizing via the `EXT4_IOC_RESIZE_FS` `ioctl()` command. This operation does not (yet) work with filesystems using the "bigalloc" or "meta_bg" features. 

  * The `/proc` filesystem has a new subdirectory for each process called `map_files`; it contains a symbolic link describing every file-backed mapping used by the relevant process. This feature is one of many needed to support the desired checkpoint/restart feature. 

  * `/proc` also supports a couple of new mount options. When mounted with `hidepid=1`, `/proc` will deny access to any process directories not owned by the requesting process. With `hidepid=2`, even the existence of other processes will be hidden. The default (`hidepid=0`) behavior is unchanged. The other new option (`gid=N`) provides an ID for a group that is allowed to access information for all processes regardless of the `hidepid=` setting. 

  * New drivers: 

    * **Systems and processors** : AppliedMicro APM8018X PowerPC processors, Numascale NumaChip systems, IBM Currituck (476fpe) boards, and NVIDIA Tegra30 processors. 

    * **Input** : TI TCA8418 keypad decoders, Wacom Intuos4 wireless tablets, EETI eGalax multi-touch panels, GPIO-connected tilt switches, Sharp GP2AP002A00F I2C Proximity/Opto sensors, and PIXCIR I2C touchscreens. 

    * **Miscellaneous** : P7IOC PowerPC I/O hubs, Dialog Semiconductor DA9052/53 PMIC devices, SiRF SoC Platform Serial ports, Analog Devices AD5421, AD5764, AD5744, and AD5380 digital to analog converters, GE PIO2 VME Parallel I/O cards, OMAP 2/3/4 displays, OMAP "Tiling and Isometric Lightweight Engine for Rotation" devices, Dialog DA9052/DA9053 regulators, VIA hardware watchdog timers, and TI TCA6507 I2C LED controllers. 

    * **Network** : Calxeda 1G/10G XGMAC Ethernet interfaces and ISA-based CC770 CAN controllers. 

    * **USB** : Marvell USB OTG transceivers and Marvell EHCI host controllers. 

    * **Graduations** : Microsoft's Hyper-V virtual network driver and the gma500 graphics driver have moved out of staging into the mainline.  

Changes visible to kernel developers include: 

  * A reworked version of the DMA buffer sharing API has been merged; this API has been described in [a separate article](<https://lwn.net/Articles/474819/>). 

  * The "memblock" low-level memory allocation API has been substantially reworked. 

  * Quite a few VFS interfaces have been changed to use the `umode_t` type for file mode bits. 

  * Also in the VFS: most of the members of `struct vfsmount` have been moved elsewhere (to a containing `struct mount`) and hidden from filesystem code. A number of callbacks in `struct super_operations` (specifically: `show_stats()`, `show_devname()`, `show_path()` and `show_options()`) now take a pointer to `struct dentry` instead of `struct vfsmount`. 

  * The [pin control subsystem](<https://lwn.net/Articles/468759/>) has gained a new configuration interface. 

  * Boolean module parameters have traditionally allowed the underlying module variable to be of either `bool` or `int` type. That tolerance is coming to an end with 3.3, where non-`bool` types will generate a warning; the plan is apparently to change those warnings to fatal compilation errors in the 3.4 cycle. A lot of modules have seen type changes for their parameters in preparation for the new regime. 

  * The "system device" type has been removed from the kernel; all instances have been converted to regular devices instead. See [this article](<https://lwn.net/Articles/474976/>) for more information. 

The merge window can be expected to remain open through approximately January 18.
