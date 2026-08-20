---
title: The first half of the 6.1 merge window
url: https://lwn.net/Articles/910312/
date: "October 7, 2022"
category: "Releases-6.1"
author: "By Jonathan Corbet October 7, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 7, 2022

The 6.1 merge window is well underway: since it opened, 5,752 non-merge changesets have been pulled into the mainline repository. That is approximately half of the work that had piled up in linux-next and marks a good time to look at what has been merged so far. Some long-awaited core changes have landed for the next kernel release, but there are likely to be more significant changes to come. 

Interesting changes merged so far include: 

#### BPF

  * The ability to create [destructive BPF programs](<https://lwn.net/Articles/901284/>) has been added. The initial proposal included a `panic()` function, but what was finally merged was a `crash_kexec()` kfunc instead. This is a lower-level entry point that does not do all of the cleanup work that `panic()` tries to handle. 
  * There is support in BPF for iterating through control groups. This feature is undocumented; some information can be found in [this commit](<https://git.kernel.org/linus/d4ccaf58a847>). 
  * The [BPF-specific memory allocator](<https://lwn.net/Articles/899274/>) has been merged. 
  * There is a new "user ring buffer" BPF map type for fast data transfer between a BPF program and user space. Little documentation exists, but [this merge message](<https://git.kernel.org/linus/c12a03766788>) has an overview. 
  * BPF task iterators can now iterate through the resources of a single thread; [this merge message](<https://git.kernel.org/linus/aa55dfd3783f>) appears to be the only documentation that exists. 

#### Core kernel

  * Experimental support for the Rust programming language has been [pulled](<https://git.kernel.org/linus/8aebac82933f>) into the mainline. Following the [decisions made](<https://lwn.net/Articles/908347/>) at the recently concluded kernel maintainers summit, this is a minimal version of the patch set, without the drivers that have been implemented in Rust so far. [This documentation commit](<https://git.kernel.org/linus/d07479b211b7>) contains more information, and [some small sample modules](<https://git.kernel.org/linus/e4fc6580b079>) are available as well. 
  * Support for the a.out executable format [was disabled](<https://lwn.net/Articles/888741/>) in the 5.18 release. Nobody has complained, so now that support has been removed entirely from the kernel. 

#### Filesystems and block I/O

  * The erofs filesystem has gained the ability to share duplicated data across filesystems. 
  * The [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) system call can now return information on whether a given file can support direct I/O and, if so, what the alignment requirements are. There is [a man-page update](<https://lwn.net/ml/linux-fsdevel/20220722074229.148925-1-ebiggers@kernel.org/>) describing the new API, and some information can be found in [this changelog](<https://git.kernel.org/linus/825cf206ed51>) as well. 

#### Hardware support

  * **Graphics** : MediaTek mt8195 DisplayPort units. 
  * **Hardware monitoring** : Microchip EMC2305 and compatible EMC2301/2/3 fan controllers, Analog Devices MAX31760 fan-speed controllers, and TEXAS tps546d24 buck converters. 
  * **Miscellaneous** : Microchip PCI1XXXX's I2C interfaces, MediaTek MT6331 and MT6332 power-management ICs, Nuvoton NCT6116 GPIO controllers, the AMD platform management framework, MediaTek MT6370 backlight controllers, Broadcom STB memory controllers, and Alibaba T-Head Yitian 710 DDR Sub-system Driveway performance-monitoring units. 
  * **Networking** : Wangxun GbE PCI Express adapters, Microsemi Ocelot controllers, and Analog Devices ADIN1110 MAC-PHYs. 
  * **Sound** : Texas Instruments SRC4XXX DIR/DIT and SRC codecs, Everest Semi ES8326 codecs, NXP i.MX8ULP digital signal processors, Apple Silicon MCA audio subsystems, and Intel SkyLake and KabyLake audio processors. 
  * Some support patches to enable [creating human input device (HID) drivers in BPF](<https://lwn.net/Articles/909109/>) have been merged, but the full mechanism looks like it will have to wait another cycle. 

#### Miscellaneous

  * The top-level page for the kernel's Documentation tree has been significantly rewritten to ease the process of finding something useful, especially when using a web browser. The results can be seen now in [linux-next](<https://docs.kernel.org/next/>), and will show up in the rendered mainline documentation after 6.1-rc1. 

#### Networking

  * Support for the old DECnet protocol has been removed from the kernel. The user-space API definitions remain so that programs with DECnet support will continue to compile, but they will never find a functioning DECnet network. 
  * The netlink protocol has finally [been documented](<https://git.kernel.org/linus/510156a7f0cb>) in the kernel source. The [HTML version of this manual](<https://docs.kernel.org/next/userspace-api/netlink/index.html>) is available on kernel.org. 
  * The TCP stack can now use a separate, per-namespace hash table for sockets, improving performance on systems with a lot of busy network namespaces. This table is disabled by default; [this commit](<https://git.kernel.org/linus/d1e5e6408b30>) describes how to configure it. 

#### Security-related

  * The LLVM-based control-flow-integrity mechanism for the kernel, which was merged for the 5.13 release, [has been replaced](<https://lwn.net/Articles/898040/>) with a new implementation that better suits the kernel's needs. 
  * There is a new security-module hook to control the ability to create user namespaces; this feature was merged despite [strong opposition](<https://lwn.net/Articles/903580/>) (which [is ongoing](<https://lwn.net/ml/linux-kernel/87sfk3mim9.fsf@email.froward.int.ebiederm.org/>)) from the namespace maintainer. 
  * Support for performing PKCS#7 signature verification in BPF programs has been added. Documentation has evidently been deferred for another development cycle, but [this merge message](<https://git.kernel.org/linus/66d6a4bf94f4>) gives an overview of why this feature exists and how it works. 

#### Internal kernel changes

  * The API for thermal-related devicetree initialization has been significantly changed, and drivers have been updated to match. The new API is resolutely undocumented, but some information can be found in [this commit](<https://git.kernel.org/linus/3fd6d6e2b4e8>). 

As of this writing, there are numerous major repositories that are yet to be pulled into the mainline, so there are almost certainly some big changes waiting in the wings. Tune back in after the close of the merge window (almost certainly on October 16) for a summary of the changes pulled in the second half.
