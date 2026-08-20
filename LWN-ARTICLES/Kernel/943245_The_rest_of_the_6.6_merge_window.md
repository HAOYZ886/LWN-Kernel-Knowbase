---
title: The rest of the 6.6 merge window
url: https://lwn.net/Articles/943245/
date: "September 11, 2023"
category: "Releases-6.6"
author: "By Jonathan Corbet September 11, 2023"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 11, 2023

Linus Torvalds [released 6.6-rc1](<https://lwn.net/ml/linux-kernel/CAHk-=wgfL1rwyvELk2VwJTtiLNpwxTFeFtStLeAQ-2rTRd34eQ@mail.gmail.com/>) and closed the 6.6 merge window on September 10\. At that point, 12,230 non-merge changesets had been pulled into the mainline repository, which is exactly 500 more than were pulled for 6.5 at this stage in the cycle. Over 7,000 of those changes were pulled after [our first-half summary](<https://lwn.net/Articles/942954/>) was written; they brought a fair amount of new functionality with them. Read on for an overview of those changes. 

Significant changes pulled in the second half of this merge window include: 

#### Architecture-specific

  * After [years of work and discussions](<https://lwn.net/Articles/926649/>), the Intel shadow-stack patches have finally been merged. This hardware feature hardens user-space code against stack-overwrite attacks. 
  * The PowerPC architecture now supports "hotplug SMT", where some or all of the simultaneous multithreading CPUs in a given core can be enabled or disabled at run time. 
  * Changes merged for RISC-V include support for user-space access to performance counters, the ability to locate crash kernels above the 4GB boundary, support for [LLVM-implemented forward-edge control-flow integrity](<https://lwn.net/Articles/898040/>), and support for kernel address-space layout randomization. 
  * The s390 architecture has removed the `noexec` command-line parameter, on the reasoning that there is no point in being able to disable non-executable mappings. This change mirrors [a similar removal](<https://git.kernel.org/linus/76ea0025a214>) for x86 in 5.19. 
  * The LoongArch architecture has gained support for a number of kernel-development tools, including KDB, KGDB, kcov, KFENCE, and KASAN. 

#### Core kernel

  * The new "eventfs" subsystem significantly improves the memory efficiency of the tracing subsystem. There are thousands of tracepoints in the kernel, each of which requires inodes and dentries for its representation in the filesystem. Those structures exist even if tracing is not used, wasting memory. Eventfs will allocated the needed structures only when they are needed, eliminating that waste. 
  * The `/proc/_pid_ /smaps` file now includes information on how well kernel samepage merging (KSM) is working to reduce memory usage. 

#### Filesystems and block I/O

  * The overlayfs filesystem has gained the ability to use fs-verity data to ensure the integrity of data supplied by lower-level filesystems; this is part of [a larger effort](<https://lwn.net/Articles/933616/>) to get overlayfs to provide the features needed by the [composefs](<https://lwn.net/Articles/922851/>) filesystem. 
  * The reiserfs filesystem has been [marked as being obsolete](<https://git.kernel.org/linus/2363650b22ed>), a step toward its eventual removal. 
  * The process of merging the online repair functionality for XFS has begun, though it may be a while yet before this code is all upstream and ready for use. 
  * The NFS server now supports NFSv4 write delegations. This mechanism allows for more caching of data and metadata for a file being written to, reducing network traffic and increasing performance. 
  * The Ceph filesystem has gained [fscrypt](<https://docs.kernel.org/filesystems/fscrypt.html>) support. 
  * The ksmbd server (providing in-kernel SMB3 filesystem support) is no longer considered "experimental". 

#### Hardware support

  * **Clock** : StarFive JH7110 clocks, Renesas VersaClock 3 clock controllers, and Qualcomm IPQ5018 global clock controllers. 
  * **GPIO and pin control** : Amlogic C3 SoC pin controllers, Qualcomm SM6115 and SM8350 LPASS LPI pin controllers, and Intel Tangier pin controllers. 
  * **Graphics** : Visionox R66451 panels and Loongson display controllers. 
  * **Industrial I/O** : Allwinner D1/T113s/T507/R329 analog-to-digital converters, Murata IRS-D200 PIR sensors, and Microchip MCP4728 digital-to-analog converters. 
  * **Input** : Azoteq IQS7222D touch controllers and Azoteq IQS7210A/7211A/E trackpad/touchscreen controllers. 
  * **Media** : Generic I2C address translators, TI DS90UB960 FPD-Link III deserializers, TI DS90UB913 and DS90UB953 FPD-Link III serializers, Dongwoo DW9719 lens voice coils, and Intel visual sensing controllers. 
  * **Miscellaneous** : Huawei HCCS interconnect controllers, Loongson-2 SoC power management controllers, Schweitzer Engineering Laboratories SEL-3350 LED controllers, Siemens Simatic CMOS batteries, Nuvoton NPCM PECI controllers, IBM I2C responders, NXP QorIQ eFuses, Qualcomm secure QFPROMs, Google Stadia force-feedback controllers, Starfive JH7110 PCIE 2.0 PHYs, StarFive JH7110 D-PHYs, Qualcomm M31 HS PHYs, Maxim MAX735x/MAX736x I2C mux/switches, NXP PCA995x I2C-controlled LEDs, Siemens Simatic IPC BX-21A LEDs and Marvell GTI watchdog timers. 
  * **Sound** : TI TAS2781 codecs, Cirrus Logic CS35L56 amplifiers, Realtek RT1017 SDCA codecs, Awinic aw88261 amplifiers, Intel Lunarlake audio interfaces, AMD ACP5x audio interfaces, and Cirrus Logic CS42L43 codecs. **Also** : the sound layer can now manage devices provided by the industrial I/O subsystem as audio devices. 
  * **USB** : Realtek RTD USB2 and USB3 PHY transceivers, USB MIDI 2.0 gadget interfaces, and Starfive JH7110 USB 2.0 PHYs. 

#### Miscellaneous

  * The `tools/workqueue` directory includes a new tool, `wq_dump.py`, that can be used to examine the current configuration of the system's workqueues and how they map onto worker pools. See [this commit](<https://git.kernel.org/linus/7f7dc377a3b2>) for some more information. 
  * Another set of enhancements to the `perf` tool has been merged; see [this commit](<https://git.kernel.org/linus/535a265d7f0d>) for details. 

#### Security-related

  * The NFS server no longer supports the (long-deprecated) DES and triple-DES Kerberos encryption types. 

#### Virtualization and containers

  * The [VFIO subsystem](<https://docs.kernel.org/driver-api/vfio.html>) provides [a new, char-device interface](<https://git.kernel.org/linus/8b6f173a4ce4>) for controlling VFIO devices. 
  * The kernel can now support [SEV-SNP](<https://www.amd.com/content/dam/amd/en/documents/epyc-business-docs/white-papers/SEV-SNP-strengthening-vm-isolation-with-integrity-protection-and-more.pdf>) and [TDX](<https://www.intel.com/content/www/us/en/developer/articles/technical/intel-trust-domain-extensions.html>) guests on Hyper-V. 

#### Internal kernel changes

  * Unbound workqueues have gained a number of options aimed at improving performance on larger systems; see the latter part of [this article](<https://lwn.net/Articles/937416/>) and [the "affinity scopes" section](<https://kernel.org/doc/html/next/core-api/workqueue.html#affinity-scopes-and-performance>) of `Documentation/core-api/workqueue.rst` for the details. 
  * Kernel builds with `W=1` set now enable the `-Wformat-overflow`, `-Wformat-truncation`, `-Wstringop-overflow`, and `-Wrestrict` warnings. The `-Wenum-conversion` warning is now enabled unconditionally. 
  * There was an increase of 326 exported symbols during this merge window; see [this page](<https://lwn.net/Articles/944037/>) for the full list of changes. One kfunc (`bpf_map_sum_elem_count()`) was added as well. 
  * The support files for the GitLab continuous-integration tests used by the graphics subsystem have been merged. See [`Documentation/gpu/automated_testing.rst`](<https://www.kernel.org/doc/html/next/gpu/automated_testing.html>) for more information on how this testing works. 

One significant change that was _not_ merged was, once again, the bcachefs filesystem. Curious readers can see [the associated discussion](<https://lwn.net/ml/linux-kernel/20230903032555.np6lu5mouv5tw4ff@moria.home.lan/>); in short, Torvalds [was not happy](<https://lwn.net/ml/linux-kernel/CAHk-=wjUX287gJCKDXUY02Wpot1n0VkjQk-PmDOmrsrEfwPfPg@mail.gmail.com/>) that the code had never appeared in linux-next (a situation that is [being rectified](<https://lwn.net/ml/linux-fsdevel/20230910043118.6xf6jgeffj5es572@moria.home.lan/>)) and had some problems that wider testing should have found. Even Torvalds, though, said that ""any further changes are better done in-tree""; it should be possible to get this code into 6.7. 

Meanwhile, the time has come to fix as many bugs as possible and to get this code into shape for the 6.6 release. Assuming the usual schedule holds (and it always does anymore), that release can be expected on October 29 or November 5\. While it is possible that the next release, 6.7, could happen on December 31, that seems unlikely, meaning that 6.6 is almost certainly the final kernel release for 2023. Should that prove to be the case, it will probably be the long-term-support release for this year as well.
