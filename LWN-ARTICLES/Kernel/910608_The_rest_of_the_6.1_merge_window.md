---
title: The rest of the 6.1 merge window
url: https://lwn.net/Articles/910608/
date: "October 17, 2022"
category: "Releases-6.1"
author: "By Jonathan Corbet October 17, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 17, 2022

Linus Torvalds [released 6.1-rc1](<https://lwn.net/ml/linux-kernel/CAHk-=wj6y5fipM2A5kEuOO9qm5PBzUY=-m9viEahhtxT09KR_g@mail.gmail.com/>) and closed the 6.1 merge window on October 16; at that point, 11,537 non-merge changesets had been pulled into the mainline repository. That is considerably less than the 13,543 changesets pulled during the 6.0 merge window, but quantity is not everything: there were quite a few significant changes brought in this time around. Many of those were part of the nearly 5,800 changesets pulled since [our first 6.1 merge window summary](<https://lwn.net/Articles/910312/>); read on for a look at some of the work done in the latter part of this merge window. 

#### Architecture-specific

  * The Loongarch architecture has gained support for perf events, kexec, kdump, and BPF just-in-time compilation. 
  * On EFI systems, the kernel is now able to decompress and launch itself in an architecture-independent way. 

#### Core kernel

  * There is a new io_uring mode (obtained by setting up a ring with the `IORING_SETUP_DEFER_TASKRUN` flag) that causes the execution of ring-related work to be deferred until the application asks for it to be done. This deferral helps to batch up work and prevent latency problems caused by the preemption of the application at inopportune times. There is a little bit of information in [this commit](<https://git.kernel.org/linus/c0e0d6ba25f1>). 
  * The process-ID control-group controller can now remember the highest number of PIDs used, which can be consulted when deciding what the limit should be; this value is exported in a virtual file called `peak`. 
  * It is now possible for user-space processes to [request that a range of pages be "collapsed" into transparent huge pages](<https://lwn.net/Articles/887753/>). This makes huge pages somewhat less transparent, of course, but it also allows user space control over if and when this work is done. 
  * The [`/dev/userfaultfd` patches](<https://lwn.net/Articles/897260/>), which provide a way for system administrators to grant access to [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) functionality with filesystem access permissions, have been merged. 
  * Memory-tiering is the management of banks of memory with different performance characteristics; normally one wants the most heavily used pages to be resident in the fastest memory, while less-frequently used (but still useful) pages can be in slower RAM. Two memory-tiering patch sets, both described in [this article](<https://lwn.net/Articles/898766/>), have been merged for 6.1. "Hot page selection" improves the detection of busy pages languishing in slower RAM so that they can be promoted. Meanwhile, the "demotion-order" patches formalize the kernel's notion of what memory tiers mean and their relative performance, allowing better management of those tiers. 
  * The [multi-generational LRU](<https://lwn.net/Articles/894859/>) code has been merged. This work replaces the kernel's two-queue, least-recently-used mechanism with a multi-stage structure that, it is said, is better at identifying which memory pages are actually in use. Included is [a "kill switch"](<https://git.kernel.org/linus/354ed5974429>) that can be used to disable the new mechanism should it create problems on a given system. The [administrative guide](<https://git.kernel.org/linus/07017acb0601>) has some more information on how to control this feature, while the [design documentation](<https://git.kernel.org/linus/8be976a0937a>) gets into how it all works. 

For those wondering how the multi-generational LRU and tiering interact: the LRU deals mostly with the management of memory within each zone (each area of physically contiguous memory of the same type on the same node), while tiering is concerned with the movement of pages between zones. 
  * The [maple tree](<https://lwn.net/Articles/845507/>) data structure has also been merged. It is meant to be a more efficient replacement for red-black trees and, it is hoped, will also facilitate the implementation of range locking in the future. A number of core memory-management subsystems have been converted over to maple trees; with luck the only user-visible change will be improved performance. 
  * [Pressure-stall information](<https://lwn.net/Articles/759781/>) can now be tracked at the control-group level. Enabling this tracking may add a significant performance cost; [this commit](<https://git.kernel.org/linus/34f26a15611a>) has some more information. 

#### Filesystems and block I/O

  * Changes merged for Btrfs provide some significant performance improvement, including an ""orders of magnitude speed boost" for the `FIEMAP` `ioctl()`. 
  * Btrfs now supports buffered writes with io_uring. 
  * The Btrfs "send" operation has gained support for files protected with [fs-verity](<https://lwn.net/Articles/790185/>). 
  * The filesystems in user space (FUSE) subsystem has gained support for the creation of temporary files with `O_TMPFILE`. 

#### Hardware support

  * **Clock** : MediaTek MT6795 and MT8365 clock controllers, Qualcomm MSM8909 and SM6375 global clock controllers, Qualcomm SM6115 and SM8450 display clock controllers, Qualcomm SC8280XP graphics clock controllers, Rockchip RV1126 clock controllers, Spreadtrum UMS512 clocks, and Renesas Versaclock 7 clock controllers. 
  * **GPIO and pin control** : Cypress Cy8C95x0 pinctrl/gpio I2C expanders, Qualcomm SC8280XP and SM8450 LPASS LPI pin controllers, and MediaTek MT8188 pin controllers. 
  * **Industrial I/O** : Liteon LTRF216A light sensors, Richtek RTQ6056 current and power monitors, MEMSensing digital 3-axis accelerometers, Maxim max11205 analog-to-digital converters, and Bosch Sensortec BNO05 IMUs. 
  * **Input** : IBM operation panels, Richtek RT5120 PMIC power keys, and Pine64 PinePhone keyboards. 
  * **Media** : NXP i.MX DW100 dewarpers and MediaTek media data path interfaces. 
  * **Miscellaneous** : Synopsys DWC AHCI SATA controllers, Microchip LAN9662 OTP controllers, Microchip PolarFire SoC reset controllers, Aspeed HACE cryptographic engines, and Exar watchdog timers. 
  * **Multi-function device** : MediaTek MT6370 power-management ICs Richtek RT5120 power management ICs, and Rockchip RK817 PMIC battery chargers. 
  * **USB** : Sunplus SP7021 USB 2.0 PHYs and MediaTek MT6370 Type-C controllers. 

#### Miscellaneous

  * As usual, there is a long list of improvements to the `perf` tool, including improved support for AMD processors; [this merge message](<https://git.kernel.org/linus/d465bff130bf>) contains a summary. 

#### Security-related

  * Support for `O_NONBLOCK` was (accidentally) removed from `/dev/random` back in 5.6. Somebody eventually noticed and complained, so that support will return in 6.1 (and, presumably, the stable updates). 

#### Internal kernel changes

  * There is continued refactoring within the various slab allocators to increase the amount of code that can be shared among them. One behavioral change that might be noticed is that SLAB will now obtain large allocations directly from the page allocator, without keeping a separate cache around. 

The long-term goal of this work is to eventually be able to remove the SLAB and SLOB allocators, but that is still a distant prospect. 
  * There is now an interface to show how much memory is wasted due to the rounding-up of allocation sizes by `kmalloc()`; [this commit](<https://git.kernel.org/linus/6edf2576a6cc>) describes how to use it. 
  * The kernel build system now requires a minimal GNU Make version of 3.82. 
  * The x86 architecture code will now emit warnings if a kernel subsystem sets up memory that is both writable and executable. The initial plan to prohibit such mappings entirely has been deferred due to a couple of late-arriving surprises, though. 
  * The conversion of memory-management functionality to [folios](<https://lwn.net/Articles/893512/>) continues, with most of the effort this time focused on the swap code. 
  * The kernel memory sanitizer (KMSAN), briefly described in [this article](<https://lwn.net/Articles/909245/#more>), has been merged. KMSAN is focused on detecting uses of uninitialized memory in the kernel; [this documentation patch](<https://git.kernel.org/linus/93858ae70cf4>) describes its use. 
  * The `checkpatch.pl` script will now complain about uses of the deprecated `kmap()` and `kmap_atomic()` functions; code should be using `kmap_local_page()` instead. 
  * There has been a fair amount of churn related to how kernel code obtains random numbers; [this merge message](<https://git.kernel.org/linus/f1947d7c8a61>) contains the rules that describe how the internal API for random number is meant to be used. 

Now the time has come to stabilize all of that work and turn it into the 6.1 release, which is slated to become this year's long-term-support kernel. That release can be expected on December 4 or 11; given the significance of some of the changes that have been merged, it would not be surprising if this cycle were one of those that require an extra week to become ready.
