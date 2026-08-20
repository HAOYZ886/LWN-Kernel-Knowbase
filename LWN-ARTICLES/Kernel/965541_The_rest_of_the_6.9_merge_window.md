---
title: The rest of the 6.9 merge window
url: https://lwn.net/Articles/965541/
date: "March 25, 2024"
category: "Releases-6.9"
author: "By Jonathan Corbet March 25, 2024"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 25, 2024

The [6.9-rc1](<https://lwn.net/ml/linux-kernel/CAHk-=wgOw_13JuuX4khpn4K+n09cRG3EBQWufAPBWoa0GLLQ0A@mail.gmail.com/>) kernel prepatch was released on March 24, closing the merge window for this development cycle. By that time, 12,435 non-merge changesets had been merged into the mainline, making for a less-busy merge window than the last couple of kernel releases (but similar to the 12,492 seen for 6.5). Well over 7,000 of those changes were merged after [the first-half merge-window summary](<https://lwn.net/Articles/965141/>) was written, meaning that the latter part of the merge window brought many more interesting changes. 

The most significant work merged in the second half of the 6.9 merge window include: 

#### Architecture-specific

  * The Rust language is now supported on 64-bit Arm processors. 
  * It is now possible to run 64-bit Arm CPUs in LPA2 mode, which sets up a 52-bit virtual address space. 
  * The LoongArch architecture has gained support for the [ORC stack unwinder](<https://lwn.net/Articles/728339/>), the `objtool` binary tool, and kernel live patching. 
  * RISC-V systems now support the [`membarrier()`](<https://man7.org/linux/man-pages/man2/membarrier.2.html>) system call. 

#### Core kernel

  * [Weighted interleaving](<https://lwn.net/Articles/948037/>) has been merged, providing better control over how pages are placed on heterogeneous-memory systems. 
  * Basic continuous-PTE support has been added for 64-bit Arm processors. This feature, described in [this article](<https://lwn.net/Articles/937239/>), improves performance by enabling better translation lookaside buffer (TLB) usage. 
  * Memory-management performance has also been improved with the merging of [a patch series](<https://lwn.net/Articles/932396/>) reducing `vmalloc()` lock contention. 
  * Function return probes can now access the arguments that the probed function was called with; this feature can help to spot changes made to those arguments. [This changelog](<https://git.kernel.org/linus/25f00e40ce79>) includes a usage example. 

#### Filesystems and block I/O

  * The device mapper "virtual data optimizer" (VDO) target has been added. 

> The dm-vdo (virtual data optimizer) target provides inline deduplication, compression, zero-block elimination, and thin provisioning. A dm-vdo target can be backed by up to 256TB of storage, and can present a logical size of up to 4PB. This target was originally developed at Permabit Technology Corp. starting in 2009. It was first released in 2013 and has been used in production environments ever since. It was made open-source in 2017 after Permabit was acquired by Red Hat. 

See [this documentation commit](<https://git.kernel.org/linus/04bf7ac646ab>) for design and usage information. 
  * The work to add online filesystem checking and repair to XFS continues, with another set of data structures now covered. 
  * The ext2 filesystem has been marked as deprecated, mostly as a result of its inability to properly represent dates after January 2038. The filesystem is not going anywhere anytime soon, and ext2 filesystems are also covered by the ext4 module, but users are being encouraged to avoid it. 
  * Initial support for [FUSE passthrough](<https://lwn.net/Articles/932060/>) has been merged. This feature allows I/O to files served by a user-space FUSE server to be handled directly by the kernel, without the need to involve that server. The result can be a significant performance increase in some cases. For 6.9, passthrough mode only supports privileged servers; expanding availability is a task for a future release. 

#### Hardware support

  * **Clock** : R-Car V4M clocks, Qualcomm X1E80100 clock controllers, and Qualcomm SC7180 modem clock controllers. 
  * **GPIO and pin control** : R-Car V4M (R8A779H0) pin controllers and Awinic AW9523/AW9523B I2C GPIO expanders. 
  * **Graphics** : Boe TH101MB31UIG002-28A panels, Novatek NT36672E DSI panels, Freescale i.MX8MP HDMI PVI bridges, Freescale i.MX8MP HDMI-TX bridges, Himax HX83112A-based DSI panels, and Renesas RZ/G2L display units. 
  * **Hardware monitoring** : Analog Devices LTC4282 high-current hot-swap controller I2C interfaces, Microsoft Surface Pro 9 fan monitors, Monolithic Power MPQ8785 step-down controllers, Amphenol ChipCap 2 relative humidity and temperature sensors, NZXT Kraken X53/X63/X73 and Z53/Z63/Z73 coolers, ASUS ROG RYUJIN II 360 fan/sensor controllers, Astera Labs PT5161L PCIe retimer temperature sensors, and ASPEED g6 PWM and Fan tach controllers. 
  * **Industrial I/O** : AMS AS7331 UV sensors, Analog Devices ADMFM2000 dual microwave down converters, Texas Instruments ADS1298 medical analog-to-digital converters, Microchip Technology PAC1934 power monitors, and Voltafield AF8133J 3-Axis magnetometers. 
  * **Input** : Goodix Berlin touchscreens. 
  * **Miscellaneous** : ChromeOS EC GPIO controllers, Maxim MAX6958/6959 7-segment LED controllers, Kinetic KTD2801 1-wire GPIO-controlled backlights, ON semiconductor NCP5623 LED controllers, MediaTek MIPI CSI CD-PHYs, Rockchip Samsung HDMI/eDP Combo PHYs, and Qualcomm MSM8909 and SM7150 interconnects. 
  * **Sound** : Qualcomm WCD9390/WCD9395 codecs and generic framer-based codecs. 
  * **USB** : ITE IT5205 Type-C USB Alt Mode Passive multiplexers and Realtek RTD USB2 and USB3 PHY transceivers. 

#### Miscellaneous

  * See [this merge message](<https://git.kernel.org/linus/1bbeaf83dd7b>) for a summary of new features added to the `perf` tool for 6.9. 
  * [User events](<https://lwn.net/Articles/889607/>) in the tracing subsystem can now export multiple formats; this is useful, for example, if a library's format changes. [This documentation commit](<https://git.kernel.org/linus/3727db1c09b4>) contains a little more information; see also [the series cover letter](<https://lwn.net/ml/linux-kernel/20240202184449.1674-1-beaub@linux.microsoft.com/>). 

#### Security-related

  * The kernel's [Integrity Measurement Architecture and Extended Verification module](<https://sourceforge.net/p/linux-ima/wiki/Home/>) (IMA/EVM) have been converted into proper Linux security modules. There should be no user-visible effects, but this change, which was enabled by the slowly proceeding [security-module stacking](<https://lwn.net/Articles/804906/>) work, simplifies the code considerably. 
  * The API for the [security-module system calls](<https://lwn.net/Articles/919059/>) merged in 6.8 has changed for better 32-bit compatibility. In theory, this change could break applications, but these system calls have only been in a released kernel for a few weeks, so the chances are that they are not yet being used. Linus Torvalds [agreed](<https://lwn.net/ml/linux-kernel/CAHk-=wjXK3ZDtCM754mQVWzcZdQd50dBU5Y+AoATWcCGCMWzgg@mail.gmail.com/>) that this change was probably safe to make. 

#### Internal kernel changes

  * The kernel's energy model [can now be updated at run time](<https://lwn.net/ml/linux-kernel/20240208115557.1273962-1-lukasz.luba@arm.com/>); this allows the kernel to, for example, take into account the effect of operating temperature on a CPU's energy efficiency. 
  * The minimum version of LLVM that can be used to build the kernel is now 13.0.1. 
  * The S/390 architecture has long used kernel virtual addresses that are the same as the underlying physical addresses, leading to a certain amount of sloppiness in distinguishing between the two address spaces. There is currently [a plan](<https://git.kernel.org/linus/f9c035492f20>) to separate the kernel's address space, meaning that said sloppiness needs to be cleaned up. Some new types have been introduced to mark physical addresses, allowing the `sparse` tool to find code that uses the wrong address type. 

#### Better luck next time

  * The Arm "WXN" mode was briefly supported. When enabled, WXN disables, at the hardware level, the ability to have memory that is both writable and executable. The kernel is able to run under this mode; whether all user-space workloads can is less certain, so it is not clear whether distributions will enable WXN by default. The `arm64.nowxn` command-line option can be used to disable WXN at boot time if need be. Unfortunately, some problems resulted and WXN support was reverted for now. 
  * An attempt was made to remove the ability to collect and report statistics from the crypto layer. [This commit](<https://git.kernel.org/linus/2beb81fbf0c0>) contains the reasoning behind this removal; in short, the feature was never documented, seemingly never used, and slows down cryptographic operations. The change ended up being reverted for now — it turned out to also remove some useful functionality — but seems sure to return in a future development cycle. 

The stabilization of all this work can be expected to take until May 12 or 19, at which point the final 6.9 kernel will be released. The earlier date would put the first half of the merge window right on top of the [Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), which seems likely to create a certain amount of maintainer stress. Between now and then, though, there will be a lot of bugs to fix and rough edges to smooth; the kernel development community will be busy, as usual.
