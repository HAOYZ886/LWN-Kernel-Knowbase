---
title: The second half of the 2.6.37 merge window
url: https://lwn.net/Articles/412459/
date: "November 1, 2010"
category: "Releases-2.6.37"
author: "By Jonathan Corbet November 1, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 1, 2010

The 2.6.37-rc1 prepatch has been [released](<http://lwn.net/Articles/412634/>), so the merge window is now closed. Nearly 3100 changesets were merged between [last week's summary](<https://lwn.net/Articles/411590/>) and the closing of the window; there were 9518 non-merge changesets merged in total for 2.6.37. The most significant user-visible changes include: 

  * The last significant big kernel lock holdout - the file locking code - has been fixed. It is now possible to build a generally useful kernel without the BKL, though quite a few older drivers still require it. 

  * Support for the [CAIF](<https://secure.wikimedia.org/wikipedia/en/wiki/CAIF>) shared memory protocol has been added. 

  * The `perf probe` command has a new `--vars` option which will cause it to list the local variables which are accessible from a given probe point. With `--externs`, global variables are listed as well. It is now possible to place probes in loadable modules. 

  * The ext4 filesystem now supports "lazy inode table initialization," an option which makes the creation of filesystems faster. Ext4 now features a reworked I/O submission path which should improve performance and scalability. 

  * "Batched discard" support has been added in the form of the new `FITRIM` `ioctl()` command. This feature allows the filesystem to tell the underlying storage device about all of the unused blocks at once. So far, this feature is only implemented by the ext4 filesystem. 

  * Much of the long-delayed Xen Dom0 (hypervisor) support has finally been merged. 2.6.37 will still not be Dom0-ready; there will be at least one more development cycle required for that; see [this summary from Jeremy Fitzhardinge](<https://lwn.net/Articles/412483/>) for the full plan. 

  * The fanotify subsystem has been re-enabled, and should be available in 2.6.37. 

  * The 9p filesystem has gained POSIX access control list support. 

  * The Speakup kernel-based screen reader has been merged into the staging tree. 

  * New drivers: 

    * **Systems and processors** : aESOP Samsung S5PV210-based Torbreck boards. 

    * **Audio** : Intel MID SST DSP devices. 

    * **Block** : Cypress Astoria USB SD host controllers, Marvell PXA168/PXA910/MMP2 SD host controllers, and ST Microelectronics Flexible Static Memory Controllers. 

    * **Miscellaneous** : Basic, memory-mapped GPIO controllers, Intel Topcliff GPIO controllers, Intel Moorestown/Medfield i2c controllers, IDT CPS Gen.2 SRIO RapidIO switches, Freescale i.MX DMA engines, ARM PrimeCell PL080 or PL081 DMA engines, Cypress West Bridge Astoria controllers, USB ENE card readers, Asahi Kasei AK8975 3-axis magnetometers, OLPC XO display controller devices, Analog Devices AD799x analog/digital converters, Winbond/Nuvoton W83795G/ADG hardware monitoring chips, Flarion OFDM usb and pcmcia modems, Maxim MAX8952 and MAX8998 Power Management ICs, National Semiconductors LP3972 PMIC regulators, and Broadcom BCM63xx hardware watchdogs. 

    * **Network** : Intel Topcliff platform controller hub CAN interfaces, Technologic Systems TS-CAN1 PC104 peripheral boards, SBE wanPMC-2T3E3 interfaces, RealTek RTL8712U (RTL8192SU) Wireless LAN NICs (replaces older rtl8712 driver), Atheros AR6003 wireless interface controllers, Beeceem USB Wimax adapters, and Broadcom bcm43xx wireless chipsets. 

    * **Video4Linux2** : remotes using the RC-5 (streamzap) protocol, Konica chipset-based cameras, Sharp IX2505V silicon tuners, LME2510 DM04/QQBOX USB DVB-S boxes, Samsung s5h1432 demodulators, Several new Conexant cx23417-based boards, Nuvoton w836x7hg consumer infrared transceivers, OmniVision OV6650 sensors, OMAP1 camera interfaces, Siliconfile SR030PC30 VGA cameras, Sony imx074 sensors, and VIA integrated chipset camera controllers.  

Changes visible to kernel developers include: 

  * There have been, once again, significant changes to the Video4Linux2 driver API. The new "mediabus" layer adds flexibility for dealing with complex devices, but also complicates simpler drivers somewhat. The videotext/teletext API, long unused, has been removed. 

  * The `file_system_type` structure has a new `mount()` function which is meant to replace `get_sb()`. 

Now the stabilization period begins; the final 2.6.37 release will almost certainly happen in January.
