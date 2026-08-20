---
title: Merged for 2.6.28
url: https://lwn.net/Articles/302754/
date: "October 15, 2008"
category: "Releases-2.6.28"
author: "By Jonathan Corbet October 15, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 15, 2008

As of this writing, 4193 non-merge changesets have been incorporated for the 2.6.28 kernel. In other words, this merge window is just beginning, having merged probably less than half of the patches which will eventually find their way into the mainline. What we see so far are a lot of drivers and incremental improvements, but not many major changes. 

User-visible changes for 2.6.28 include: 

  * There are new drivers for Analog Devices SSM2602, AD1882A and AD1980 codecs, Freescale MPC5200 I2S audio devices, Texas Instruments TLV320AIC26 codecs, Tascam US-122L USB Audio/MIDI interfaces, Wolfson Micro WM8580, WM8900, WM8903, and WM8971 audio devices, Blackfin SPORT peripheral interface controllers, NVIDIA HDMI HD-audio codecs, Toshiba RBTX4939 MIPS boards, Atheros L2 10/100 network adapters, Cisco 10G Ethernet adapters, JMicron JMC250 chipset-based network adapters, QLogic QLGE 10Gb Ethernet adapters, SMSC LAN95XX based USB 2.0 10/100 ethernet devices, AFEB9260 ARM-based boards (an open source board design), Arcom/Eurotech VIPER boards, AT91SAM9X watchdog devices, ITE IT8716, IT8718, IT8726, and IT8712 Super I/O watchdogs, W83697UG/W83697UF watchdog devices, TLV320AIC23 codecs, Micron MT9M111 camera chips, Magic-Pro DMB-TH tuners, Afatech AF9015 and AF9013 DVB-T USB2.0 receivers, Conexant cx24116/cx24118 tuners, DVB cards based on SDMC DM1105 PCI chip, Silicon Laboratories SI2109/2110 demodulators, ST STB6000 DVBS Silicon tuners, numerous Fujifilm FinePix cameras, ALi video camera controllers, WM8400 AudioPlus HiFi codecs, and SGS-Thomson M48T35 Timekeeper RAM chips. 

  * Support for the old Sun 4 architecture and ColdFire serial ports has been removed. 

  * There is a new sysfs file (`unload_heads`) which can be used by a user-space process to tell an ATA disk to retract its heads and prepare for an impact. When used in conjunction with an accelerometer, this feature could be used to attempt to preserve a disk in a falling laptop. 

  * Improved support for `ptrace()` \- and support for precise event-based sampling in particular - has been added for the x86 architecture. 

  * The crypto subsystem has gained support for deterministic ANSI X9.31 A.2.4 pseudo-random number generation. 

  * The SMACK security module can now be configured to enforce mandatory access control rules on privileged processes. 

  * There is a script which can be used to generate a minimal "dummy" policy for SELinux. The smallest workable policy, it seems, is 587 lines long. 

  * Some sound devices can detect the presence of audio devices on input and output jacks. The ALSA layer now allows drivers for those devices to register those jacks and report the presence of devices attached to sound cards through the input layer. 

  * Work with multiqueue networking continues; 2.6.28 will include the ability to associate a separate queueing discipline with each internal packet queue. 

  * The [wireless regulatory compliance](<http://lwn.net/Articles/237104/>) subsystem has been merged. 

  * The kernel now supports the Phonet packet protocol used by Nokia cellular modems. See [networking/phonet.txt](<https://lwn.net/Articles/303274/>) in the kernel documentation directory for more information. 

  * Also added to core networking is support for the Distributed Switch Architecture protocol, with initial support for a number of Marvell switch chips. 

  * The netfilter layer has been augmented to support network namespaces. 

  * The ext4 system has lost the "ext4dev" name; this is a signal that the developers are getting ready to declare it ready for production use. Ext4 has also gained a set of static tracepoints for use with SystemTap or other tracing tools. 

  * The [FIEMAP](<http://lwn.net/Articles/260795/>) `ioctl()` for extent mapping has been added. 

  * Xen has added CPU hotplugging support. 

  * Version 4 of the rpcbind protocol is now supported; this enables the kernel to offer RPC services via IPv6. 

  * The OCFS2 filesystem has gained a number of features, including POSIX locks, extended attributes, and use of the JBD2 journaling layer. 

Changes visible to kernel developers include: 

  * Discard request and request timeout handling have been added to the block layer; a number of other internal API changes have been made as well. See [this article](<http://lwn.net/Articles/303270/>) for details. 

  * Video4Linux2 drivers no longer have their `open()` function called with the big kernel lock held. The `lock_kernel()` calls have been pushed down into individual drivers within the mainline tree; external drivers will need to be fixed. 

The merge window is likely to remain open until approximately October 24.
