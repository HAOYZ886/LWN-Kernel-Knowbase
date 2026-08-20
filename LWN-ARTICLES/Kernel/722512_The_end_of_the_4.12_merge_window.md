---
title: The end of the 4.12 merge window
url: https://lwn.net/Articles/722512/
date: "May 14, 2017"
category: "Releases-4.12"
author: "By Jonathan Corbet May 14, 2017"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 14, 2017

Linus Torvalds released the [4.12-rc1](<https://lwn.net/Articles/722660/>) prepatch and closed the merge window on May 13 — a move that may have surprised maintainers who were waiting until the last day to get their final pull requests in. Let that be a lesson to all: one should not expect to have pull requests honored on Mother's Day. Below is a summary of the changes merged since [the May 10 merge-window summary](<https://lwn.net/Articles/722183/>). 

In the end, 12,920 non-merge changesets found their way into the mainline during the merge window; about 1,000 since the previous summary was written. As expected, that was enough to make the 4.12 merge window the second-busiest ever. The most significant changes found in that last 1,000 changesets include: 

  * The [TEE framework](<https://lwn.net/Articles/717125/>) has been merged; this mechanism allows the kernel to support trusted execution environments on processors (such as ARM CPUs with TrustZone) with that capability. 

  * Support for parallel NFS (pNFS) on top of object storage devices has been removed; that support is unused and has been unmaintained for some time. 

  * The building of the old Open Sound System audio drivers has been disabled for the 4.12 release. In the absence of screaming, those drivers will likely be removed in the relatively near future. They are poorly maintained and nearly (if not completely) unused, but the driving motivation behind this change at the moment is the desire to [eliminate the many `set_fs()` calls](<https://lwn.net/Articles/722267/>) found in those drivers. 

  * New hardware support includes: Mediatek MT6797 clocks, HiSilicon Hi655x clocks, Allwinner PRCM clock controllers, Motorola CPCAP PMIC realtime clocks, STMicroelectronics STM32 Quad SPI controllers, Broadcom BCM2835 and BCM470xx thermal sensors, Dialog Semiconductor DA9062/DA9061 thermal sensors, Powers AXP20X battery power supplies, and PlayStation 1/2 joypads via SPI interfaces. 

  * There is [a new set of macros](<https://git.kernel.org/linus/bf616d21f41174389c6d720ae21bf40f154474c8>) that allow the marking of boot-time and module parameters that modify hardware behavior. Numerous drivers have been patched to make use of these macros. The intent is to disallow a user from changing these parameters on systems where UEFI secure boot is in use, but that mechanism has not yet been merged. 

  * Synchronous read-copy-update grace periods may now be used anywhere in the kernel's boot process; see [this article](<https://lwn.net/Articles/716148/>) for the details. 

The time to test and stabilize the 4.12 kernel has begun; if all goes according to the usual schedule, the final release can be expected in early July.
