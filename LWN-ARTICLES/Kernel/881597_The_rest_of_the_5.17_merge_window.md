---
title: The rest of the 5.17 merge window
url: https://lwn.net/Articles/881597/
date: "January 24, 2022"
category: "Releases-5.17"
author: "By Jonathan Corbet January 24, 2022"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 24, 2022

Linus Torvalds [released 5.17-rc1](<https://lwn.net/ml/linux-kernel/CAHk-=whUYZvNVtNc4Xftd8aDpZzuwswBtHG4fCYQ2AC3twh_5Q@mail.gmail.com/>) and closed the 5.17 merge window on January 23 after having pulled just over 11,000 non-merge changesets into the mainline repository. A little over 4,000 of those changesets arrived after [our first-half merge-window summary](<https://lwn.net/Articles/880909/>) was written. Activity thus slowed down, as expected, in the second half of the merge window, but there still a number of significant changes that made it in for the next kernel release. 

The more interesting changes merged for 5.17 include: 

#### Core kernel

  * It is now possible to assign names to "private" anonymous-memory areas; this feature is useful when trying to optimize application memory use. See [this article](<https://lwn.net/Articles/867818/>) and [this commit](<https://git.kernel.org/linus/9a10064f5625>) for more information. 
  * The kernel is now able to decompress kernel modules within the kernel itself, rather than relying on user space to do that work. This feature was added to allow the [LoadPin security module](<https://lwn.net/Articles/682302/>) to maintain the connection between the module code on the storage device and what gets loaded into memory. 

#### Filesystems and block I/O

  * The fanotify subsystem supports a new event type called `FAN_RENAME`. It is meant as a replacement for the old `FAN_MOVED_FROM` and `FAN_MOVED_TO` events that captures an entire rename operation in a single event; the old event types are, of course, still supported. 
  * The FUSE filesystem was running out of initialization flags, so the `FUSE_INIT` operation has been extended with a new set of flags. While known FUSE implementations have been shown to work with the extended flags, the possibility of a user-space surprise exists. Should one turn up, this change will probably need to be reverted and replaced with something more complicated. 
  * The "fscache" subsystem, which handles caching for network filesystems, has been torn out and rewritten from scratch. For the most part, there should be few, if any, user-visible changes resulting from this work. [This merge commit](<https://git.kernel.org/linus/8834147f9505>) provides an overview of this work and why it was done. 
  * The old `/sys/class/dax` compatibility option has been removed; this change, too, is not known to break any actually deployed systems but may be reverted if somebody complains. 
  * There is a new syntax for Ceph filesystem mounts that addresses a number of problems; see [this documentation commit](<https://git.kernel.org/linus/e1b9eb50763d>) for an overview of what it looks like. 

#### Hardware support

  * **Clock** : R-Car Gen4 and S4-8 clock controllers, TI TPS68470 PMIC clock controllers, Allwinner D1 clock-control units, Qualcomm MSM8956/76, SDX65 and SM8450 global clock controllers, Toshiba Visconti5 clock controllers, MediaTek MT7986 clock controllers, Nintendo realtime clocks, and Sunplus SP7021 realtime clocks. 
  * **Industrial I/O** : Analog Devices AD7293 power amplifier current controllers, Analog Devices AD74412R/AD74413R I/O controllers, Analog Devices ADMV8818 high-pass/low-pass filters, Analog Devices AD3552R digital-to-analog converters, and Analog Devices ADMV1013 microwave upconverters. 
  * **Miscellaneous** : Mediatek MT6360 PMIC LED controllers, Mediatek SPMI controllers, MStar MSC313E timers, Xilinx AMS system monitors, Renesas R-Car Gen3 realtime processors, and Qualcomm MSM8996, QCM2290, and SM8450 interconnect providers. 

Also, Intel's "platform firmware runtime update" driver has been merged; it allows parts of the system firmware to be updated without the need to restart the system. Some more information can be found in [this document](<https://uefi.org/sites/default/files/resources/Intel_MM_OS_Interface_Spec_Rev100.pdf>). 
  * **PHY** : Qualcomm eDP PHYs, Microchip lan966x Ethernet PHYs, Intel Thunder Bay eMMC PHYs, and Freescale i.MX8M PCIE PHYs. 
  * **Sound** : Mediatek MT8195 audio DSPs, AMD Renoir audio DSPs, Texas Instruments TLV320ADC3001/3101 audio codecs, and Asahi-Kasei AK4375 codecs. 
  * **Watchdog** : Realtek Otto MIPS watchdog timers, Renesas RZ/G2L WDT watchdog timers, and Apple SoC watchdog timer. 

#### Miscellaneous

  * The `perf ftrace` command has a new `latency` subcommand to generate latency histograms, optionally using a BPF script for the data collection. This feature was brought over from the [BCC tools](<https://github.com/iovisor/bcc>). 
  * The "RTLA" set of tools for realtime system analysis has been added. These tools include `osnoise` (measures operating-system interference with task execution) and `timerlat` (measures timer latency). 

#### Security-related

  * [Mitigation for straight-line speculation vulnerabilities](<https://lwn.net/Articles/877845/>) has been merged; building a kernel with this protection will require GCC 12, once it is released. 
  * The new "page-table check" feature adds extra checks when new entries are added to process page tables. It is able to detect certain types of corruption and bring down the system before something worse happens; some information can be found in [this commit](<https://git.kernel.org/linus/df4e817b7108>). 

#### Internal kernel changes

  * The next batch of [folio](<https://lwn.net/Articles/849538/>) patches has been merged. This work converts most of the page-cache code to use folios, but does not yet quite complete the job. Various APIs related to the page cache have changed as a result. The XFS filesystem has also received a set of folio-conversion patches. 
  * The new `kthread_run_on_cpu()` helper creates a new kernel thread on the specified CPU and starts it running. 
  * `vmalloc()` now supports the `__GFP_NOFAIL` flag for those cases where an allocation absolutely cannot be allowed to fail. Developers considering using this flag should keep in mind Andrew Morton's [admonition](<https://lwn.net/ml/linux-kernel/20211127160043.1512b4063f30b4d043b37420@linux-foundation.org/>) that ""every use is a sin"", though. The `GFP_NOFS` and `GFP_NOIO` flags are also now supported with `vmalloc()`. 
  * The new `make mod2noconfig` build target creates a new kernel configuration where all subsystems formerly built as modules are, instead, disabled. 
  * The oldest LLVM release that can be used to build the kernel has been increased to 11.0.0. [This commit](<https://git.kernel.org/linus/df05c0e9496c>) explains the motivation for that change. 
  * The "cleancache" mechanism, which was [merged in 2010](<https://lwn.net/Articles/386090/>), has been removed. It seems that there are no users of cleancache at this point, and thus no reason to keep it around. Most of the associated "frontswap" mechanism is also gone, and it is no longer possible to select frontswap as a separate configuration option. 

The time has now come to stabilize all of that work that has just entered the mainline. That process can be expected to unfold over the next seven or eight weeks, with the final 5.17 release happening on March 13 or 20.
