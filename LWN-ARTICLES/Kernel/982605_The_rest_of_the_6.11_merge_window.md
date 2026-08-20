---
title: The rest of the 6.11 merge window
url: https://lwn.net/Articles/982605/
date: "July 29, 2024"
category: "Releases-6.11"
author: "By Jonathan Corbet July 29, 2024"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 29, 2024

The release of [6.11-rc1](<https://lwn.net/ml/all/CAHk-=wiyNokz0d3b=GRORij=mGvwoYHy=+bv6m2Hu_VqNdg66g@mail.gmail.com/>) marked the end of the 6.11 merge window on July 28\. By that time, 12,102 non-merge changesets had been pulled into the mainline repository; about 8,000 of those came in after [the first-half summary](<https://lwn.net/Articles/982034/>) was written. Quite a few significant changes were to be found in those changesets; there is also one big change that did not make it. 

Some of the most interesting changes pulled into the mainline in the latter part of the 6.11 merge window include: 

#### Architecture-specific

  * There is a new implementation of the AES-GCM cipher for x86-64 systems. The resulting performance improvement is always welcome, but this contribution is also noteworthy for the extensive documentation of the code and how it works; see [this commit](<https://git.kernel.org/linus/b06affb1cb58>) for the whole thing. 
  * Support for PowerPC 40x platforms has been removed. 
  * Memory hotplugging support has been added for RISC-V systems. 
  * The [kernel memory sanitizer (KMSAN)](<https://docs.kernel.org/dev-tools/kmsan.html>) is now supported on s/390 systems. 
  * RISC-V systems have gained support for the [STACKLEAK](<https://lwn.net/Articles/764325/>) security feature. 

#### Core kernel

  * There is a new system call, `uretprobe()`, that is currently only implemented on x86-64 systems. Its job is to optimize the handling of [uretprobes](<https://docs.kernel.org/trace/uprobetracer.html>) (function-return tracepoints dynamically inserted into a user-space process); nothing good will come if a process tries to call it directly. See [this commit](<https://git.kernel.org/linus/ff474a78cef5>) for a description of how it works and [this LSFMM+BPF report](<https://lwn.net/Articles/978335/>) for some more information on the use case that motivated it. 
  * The single-user limit for the [function-graph tracer](<https://docs.kernel.org/trace/ftrace.html#function-graph-tracer>) has been removed. 
  * The new `reserve_mem=` command-line parameter can be used to set aside a named region of memory; it is intended to be used, in conjunction with the new `ramoops.mem_name=` parameter, with the [pstore](<https://docs.kernel.org/admin-guide/pstore-blk.html>) subsystem. See [this commit](<https://git.kernel.org/linus/1e4c64b71c9b>) and [this commit](<https://git.kernel.org/linus/d9d814eebb1a>) for details. 
  * The control-group memory controller now supports a `swappiness=` parameter. It allows proactive-reclaim algorithms to set the balance between swapping (reclaiming anonymous pages) and reclaiming file-backed pages. See [this commit](<https://git.kernel.org/linus/68cd9050d871>) for details. 
  * There is a new `ioctl()` interface to the memory-layout information found in `/proc/_PID_ /maps` that allows for more efficient access. See [this commit](<https://git.kernel.org/linus/c10cb9148e51>) for minimal documentation, [this commit](<https://git.kernel.org/linus/ed5d583a88a9>) for a long description of the work and the use case it addresses, and [this commit](<https://git.kernel.org/linus/81510a0eaa69>) for test code showing how it is used. 

#### Filesystems and block I/O

  * The Btrfs filesystem recognizes a couple of new values (`ignoremetacsums` and `ignoresuperflags`) for the `rescue=` mount option; they are meant to assist in the recovery of filesystems that may have been partially converted by user-space tools (those that do checksum conversion in particular). 
  * The way that Btrfs does block-group reclaim has been improved in the hope of avoiding situations where free space is unavailable even though the filesystem is not full. See [this commit](<https://git.kernel.org/linus/f5ff64ccf7bb>) for a description of how the new dynamic reclaim threshold works. 
  * The ntfs3 filesystem has gained support for the "compressed" and "immutable" attributes on files. 

#### Hardware support

  * **Clock** : Amlogic C3 PLL clock controllers, Amlogic C3 peripherals clock controllers, Qualcomm SM7150 display clock controllers, Qualcomm SM7150 camera clock controllers, Qualcomm SM7150 video clock controllers, Qualcomm QCA8386 and QCA8084 NSS clock controllers, Qualcomm QCM2290 graphics clock controllers, Sophgo SG2042 PLL clock controllers, and T-HEAD TH1520 AP clock controllers. 
  * **GPIO and pin control** : NXP IMX91 pin control units, Nuvoton MA35 pin and GPIO controllers, and Qualcomm SM4250 LPASS LPI pin controllers. 
  * **Graphics** : Lincoln Technologies lcd197 panels, Ilitek ILI9806E-based panels, and STMicroelectronics LVDS display interface transmitters. 
  * **Industrial I/O** : Analog Devices AD7194 and AD7380 analog-to-digital converters, Vishay VEML6040 RGBW light sensors, ScioSense ENS160 multi-gas sensors, Texas Instruments ADS1119 analog-to-digital converters, and MediaTek MT6359 PMIC analog-to-digital converters. 
  * **Input** : Huion Inspiroy 2S and Dial 2 tablets, XP-PEN Deco Mini 4 tablets, and Cirrus Logic CS40L50 haptic drivers. 
  * **Media** : Sony IMX283 image sensors, Maxim MAX96717 GMSL2 serializers, Maxim MAX96714 GMSL2 deserializers, Imagination E5010 JPEG encoders, GalaxyCore gc08a3 sensors, and Raspberry Pi PiSP image signal processors. 
  * **Miscellaneous** : Lenovo Yoga C630 embedded controllers, ChromeOS EC LED controllers, Marvell 88PM886 power-management ICs, Qualcomm PM8008 PMIC regulators, ROHM BD96801 power-management ICs, Texas Instruments LM3509 backlight controllers, Silergy SY7802 flash LED controllers, Texas Instruments LP5569 LED controllers, Marvell Octeon data-processing units, KEBA CP500 system FPGAs, MediaTek DVFSRC EMI interconnects, Qualcomm MSM8953 interconnects, Marvel Octeon CN10K DMA packet interfaces, StarFive PCIe host controllers, Microchip LAN966x outbound interrupt controllers, Lenovo Yoga C630 battery controllers, Maxim MAX17201/MAX17205 fuel gauges, Qualcomm CPUCP mailbox controllers, and STMicroelectronics STM32 DMA3 controllers. 
  * **PHY** : StarFive JH7110 D-PHYs, Freescale i.MX8QM HSIO PHYs, and Airoha PCIe-PHYs. 
  * **Sound** : Senarytech HD-audio codecs, Realtek RT1320 codecs, Everest Semi ES8311 codecs, Qualcomm WCD9370/WCD9375 codecs, Realtek rt1318 audio amplifiers, Cirrus Logic CS530x analog-to-digital converters, Asahi Kasei AK4619 audio codecs, and NXP LPC32XX audio subsystems. 
  * **USB** : Lenovo Yoga C630 USCI interfaces. 

#### Miscellaneous

  * The `perf` tool has a number of new features; see [this changelog](<https://git.kernel.org/linus/68b59730459e>) for the full list. 
  * A small set of [Rust abstractions for firmware loading](<https://git.kernel.org/linus/de6582833db0>) has been merged. There are still many pieces needed before nontrivial drivers can be written in Rust, but the list is slowly getting smaller. Other merged abstractions include `uaccess` (user-space access) and `page` (for working with `struct page`). 
  * While the minimum version of Rust needed to build kernel code remains 1.78.0, it is now also possible to use versions 1.79.0 or 1.80.0 as well. The plan is to broaden the range of applicable compiler versions over time. 

#### Security-related

  * The [dedicated bucket slab allocator](<https://lwn.net/Articles/965837/>) has been merged; it is intended to provide protection against heap-spraying attacks. 
  * The generation of random data will be greatly accelerated by the new [vDSO implementation of `getrandom()`](<https://lwn.net/Articles/980447/>). The implementation has changed somewhat since that article was written, though; see [this update](<https://lwn.net/Articles/983186/>) for the details. Among other things, this work has added a new `MAP_DROPPABLE` flag for [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) that requests memory that can be dropped by the kernel if need be. 

#### Virtualization and containers

  * There is now initial support for AMD's [SEV-SNP](<https://www.amd.com/en/developer/sev.html>) secure encrypted virtualization mechanism in KVM, built on top of the architecture-level support added earlier in the merge window. 
  * There is a new KVM `ioctl()` call to pre-populate a guest's memory before it runs; see [this commit](<https://git.kernel.org/linus/9aed7a6c0b59>) for documentation. 

#### Internal kernel changes

  * The minimum version of GNU Make needed to build the kernel has been raised to 4.0. 
  * There were 118 exported symbols removed from 6.11, and 464 additions, for a net growth of 346; there are also six new kfuncs. See [this page](<https://lwn.net/Articles/983762/>) for the full list. 

On June 11, Linus Torvalds [announced](<https://lwn.net/Articles/978007/>) that the [extensible scheduler class](<https://lwn.net/Articles/922405/>) ("sched_ext") would be merged for 6.11. After that came [a strongly worded discussion](<https://lwn.net/ml/all/87ed8sps71.ffs@tglx/>) about whether that step was justified, or whether sched_ext needed more work first. The conclusion, it would seem, is that another development cycle (at least) will be taken toward improving this code; sched_ext was _not_ merged for 6.11. There have been some more constructive conversations about this code since then, though, so it seems likely that it will get there eventually. 

Meanwhile, the work that _was_ merged will need to be stabilized over the next seven or eight weeks. The most likely date for the 6.11 kernel to be released is September 15 — just in time for the 6.12 merge window to land on top of the Open Source Summit Europe, Maintainers Summit, and Linux Plumbers Conference. But first, there will be a lot of testing and bug fixing to be done. 

(As a reminder, LWN subscribers can look into the [LWN Kernel Source Database](</ksdb/releases/v6.11/>) for a lot of details on where the code pulled for 6.11 came from).
