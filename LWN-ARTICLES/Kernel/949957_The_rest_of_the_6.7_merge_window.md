---
title: The rest of the 6.7 merge window
url: https://lwn.net/Articles/949957/
date: "November 13, 2023"
category: "Releases-6.7"
author: "By Jonathan Corbet November 13, 2023"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 13, 2023

By the time that the 6.7 merge window closed on November 12, 15,418 non-merge changesets had been pulled into the mainline kernel. That makes this one of the busiest merge windows ever; if one discounts the lengthy bcachefs development history (some 2,800 commits), though, then the patch volume is roughly in line with other recent kernels. Over 5,000 of those commits were merged after [our first-half merge-window summary](<https://lwn.net/Articles/949294/>) was written. 

Interesting changes pulled into the mainline during the second half of the 6.7 merge window include: 

#### Architecture-specific

  * The LoongArch architecture has gained support for virtualization with KVM. There is a bit of information in [this documentation commit](<https://git.kernel.org/linus/6f0257a03212>). 
  * KVM on RISC-V now supports the [Smstateen extension](<https://drive.google.com/file/d/18z_TNyR8R7JLsxiAaF9hXwWPd_vQbOqP/view>), which is intended to prevent virtual machines from using registers unknown to the hypervisor (as a result of being added by an extension the hypervisor does not support) as covert channels. KVM on RISC-V also now allows the use of the [Zicond extension](<https://github.com/riscv/riscv-zicond/releases/download/v1.0-rc2/riscv-zicond-v1.0-rc2.pdf>) (adding some conditional integer operations) in guests. 
  * It is now possible for a KVM guest on x86 systems to have up to 4,096 virtual CPUs; the maximum is now set with a kernel configuration option. 
  * RISC-V has gained support for LLVM-based [shadow call stack](<https://clang.llvm.org/docs/ShadowCallStack.html>) protection. Clang 17 or later is needed to enable this option. 

#### Core kernel

  * [Kernel samepage merging (KSM)](<https://docs.kernel.org/admin-guide/mm/ksm.html>) tries merge anonymous pages that hold the same contents to improve memory utilization. By its nature, it ends up repeatedly scanning the pages that it has failed to merge, wasting CPU time. A new "smart scan" mode tracks the pages that were unsuccessfully scanned and causes future scans to happen at a reduced frequency. Smart scanning is ~~disabled~~ enabled by default; there is a new sysctl knob (`/sys/kernel/mm/ksm/smart_scan`) that can be used to query or change its state. Some documentation can be found in [this commit](<https://git.kernel.org/linus/75d7dd4138ed>). 
  * The new `PAGEMAP_SCAN` `ioctl()` command, available with [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) file descriptors, can be used to detect writes to a range of memory. It is useful for certain gaming anti-cheat techniques and for [CRIU](<https://criu.org/Main_Page>). See [this article](<https://lwn.net/Articles/940704/>) and [this documentation commit](<https://git.kernel.org/linus/18825b8ae9a3>) for more information. 

#### Filesystems and block I/O

  * The Ceph filesystem has gained support for [ID-mapped mounts](<https://lwn.net/Articles/896255/>). 

#### Hardware support

  * **GPIO and pin control** : Nuvoton NPCM8XX pin controllers, Realtek DHC 1315E, DHC 1319D, and DHC 1619B pin controllers, and Amlogic T7 SoC pin controllers. 
  * **Industrial I/O** : Microchip MCP39xx, MCP246x, and MCP356x analog-to-digital converters, Linear Technology LTC2309 analog-to-digital converters, Kionix KX132-1211 accelerometers, and ROHM BM1390 pressure sensors. 
  * **Media** : Nuvoton NPCM video capture/encode engines, Digiteq Automotive MGB4 grabber cards, and onsemi MT9M114 sensors. 
  * **Miscellaneous** : Renesas R-Car Gen4 PCIe controllers, Xilinx DMA PL PCIe host bridges, Kinetic KTD2026/7 RGB/white LED controllers, Qualcomm SDX75 interconnect providers, Espressif ESP32 UARTs, Espressif ESP32 USB ACM gadget controllers, and SigmaStar SSD202D realtime clocks. 
  * **Sound** : Starfive JH7110 PWM-DAC digital-to-analog converters, Richtek RTQ9128 45W digital input amplifiers, Awinic aw87390 and aw88399 amplifiers, and AMD ACP6.3 audio platforms. 
  * **USB** : Realtek DWC3 controllers, Intel La Jolla Cove USB adapters, and NXP PTN36502 Type-C redrivers. 

#### Miscellaneous

  * See [this merge message](<https://git.kernel.org/linus/7ab89417ed23>) for the long list of new features and enhancements to the `perf` tool in 6.7. 

#### Security-related

  * The [Landlock security module](<https://docs.kernel.org/security/landlock.html>) has gained the ability to control network connections. See [this commit message](<https://git.kernel.org/linus/fff69fb03dde>) and [this documentation patch](<https://git.kernel.org/linus/51442e8d64bc>) for a bit more information. 
  * The [AppArmor security module](<https://docs.kernel.org/admin-guide/LSM/apparmor.html>) can now control access to io_uring and the creation of user namespaces; this feature was merged with no documentation. 
  * The kernel has gained [a new API for virtual-machine attestation](<https://lwn.net/Articles/943239/#tsm>). See [this changelog](<https://git.kernel.org/linus/70e6f7e2b985>) for some more information. 

#### Internal kernel changes

  * The memory-management's shrinker mechanism [has been reworked](<https://lwn.net/Articles/944199/>) to eliminate some locking overhead and reduce contention. Shrinkers are now all allocated dynamically, and some common operations have been made lockless. 
  * After some setbacks, the [work to improve the reliability and performance of `printk()`](<https://lwn.net/Articles/909980/>) is back. The biggest change merged for 6.7 is a new per-console locking scheme that allows for high-priority messages (a panic, for example) to take over console output from a lower-priority message. 
  * The build system will now build `perf` BPF programs by default if a suitable version of Clang is available. 
  * The old videobuf layer, long the way of managing frame buffers in the media subsystem, [has been removed](<https://git.kernel.org/linus/2a2fffb488a3>). Drivers should have moved to videobuf2 years ago. 
  * The `MAINTAINERS` file has been [updated](<https://git.kernel.org/linus/6d795e2a7df5>) to reflect the fact that a number of kernel mailing lists have moved off lists.linux-foundation.org and onto lists.linux.dev. The old list addresses work for now but may eventually stop working, so it would be wise to update address books and such to match the new reality. 

If this were a normal nine-week development cycle, the final 6.7 release would happen on December 31\. Past history suggests, though, that it is unlikely that Linus Torvalds will choose to release the kernel then, when so many developers are away from their keyboards. So a more realistic release date is January 7, 2024. Before then, though, there will surely be a lot of bugs to find and fix, as usual.
