---
title: The rest of the 6.13 merge window
url: https://lwn.net/Articles/998990/
date: "December 2, 2024"
category: "Releases-6.13"
author: "By Jonathan Corbet December 2, 2024"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 2, 2024

The 6.13 merge window closed with the release of [6.13-rc1](<https://lwn.net/Articles/1000378/>) on December 1\. By that time, 11,307 non-merge commits had been pulled into the mainline repository; about 9,500 of those landed after [our first-half merge-window summary](<https://lwn.net/Articles/998623/>) was written. There was a lot of new material in these patches, including architecture-support improvements, new BPF features, an efficient way to add guard pages to an address space, more Rust support, a vast number of new device drivers, and more. 

The most interesting changes merged during the latter half of the 6.13 merge window include: 

#### Architecture-specific

  * Support for PowerPC "Maple" platforms has been removed. According to [this commit](<https://git.kernel.org/linus/62f8f307c80e>): ""Sadly all those machines have passed into memory, and there's been no evidence for years that anyone is still using any of them"". 
  * The Loongarch architecture has gained support for realtime preemption and lazy preemption. 
  * The kernel now supports the "Smmpm", "Smnpm", and "Ssnpm" RISC-V pointer-masking extensions; [this commit](<https://git.kernel.org/linus/09d6775f503b>) contains some documentation. 

#### Core kernel

  * There is a new `kmem_cache` iterator that enables BPF programs to obtain information about the state of the slab allocator. There is no documentation, but [this merge message](<https://git.kernel.org/linus/1477d31b1c9a>) gives an overview, and [this commit](<https://git.kernel.org/linus/a496d0cdc84d>) contains a test program using this feature. 
  * The new `bpf_send_signal_task()` kfunc allows a BPF program to send a signal to another process. 
  * There is a new BPF map feature that allows threads to share some memory with a BPF scheduler (implemented with [sched_ext](<https://lwn.net/Articles/991205/>)). The idea is to share hints about the workload that will let the scheduler do a better job. The feature is undocumented; see [this merge message](<https://git.kernel.org/linus/c6fb8030b4ba>) for an overview, and [this commit](<https://git.kernel.org/linus/4579b4a4279e>) for a test program. 
  * BPF programs can now have a private stack, reducing the risk of overflows in long call chains. [This merge message](<https://git.kernel.org/linus/c1bc51f85cd6>) contains an overview of the feature and its motivation. Only x86-64 systems support this feature for now. 
  * Support has been added to allow system-call entry and exit tracepoints to incur page faults in order to read user-space parameters. 
  * There is a new [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) operation, `MADV_GUARD_INSTALL`, that installs "guard" pages in the indicated address range; an attempt to access a guard page will result in a `SIGSEGV` signal. These guard pages are a more efficient way to prevent code from running off the end of an allocation than `PROT_NONE` mappings (the only current way to do that) because it does not require the allocation of a new virtual memory area. `MADV_GUARD_REMOVE` will remove existing guard pages. See [this commit](<https://git.kernel.org/linus/662df3e5c376>) for a bit more information. 
  * The new `transparent_hugepage_shmem=` command-line parameter controls the policy for the allocation of transparent huge pages in the shmem/tmpfs filesystems; see [this commit](<https://git.kernel.org/linus/949042811117>) for some more information. 

#### Filesystems and block I/O

  * The EROFS filesystem has gained support for the `SEEK_HOLE` and `SEEK_DATA` [`lseek()`](<https://man7.org/linux/man-pages/man2/lseek.2.html>) options. 
  * As [expected](<https://lwn.net/Articles/995116/>), the reiserfs filesystem has been removed from the kernel. 
  * The F2FS filesystem has a new feature, called "device aliasing", that allows a portion of a block device to be set aside. ""Essentially, this mechanism allows a portion of the f2fs area to be temporarily reserved and used by another filesystem or for different purposes. Once that external usage is complete, the device aliasing file can be deleted, releasing the reserved space back to F2FS for its own use."" See [this commit](<https://git.kernel.org/linus/128d333f0dff>) for a documentation update. 

#### Hardware support

  * **Clock** : Virtual machine PTP clocks, MediaTek MT6735 clocks, Nuvoton Arbel BMC NPCM8XX clock controllers, Mobileye EyeQ clock controllers, Renesas VBATTB clock controllers, Qualcomm SA8775P camera, display, and video clock controllers, Qualcomm SAR2130P global and graphics clock controllers, Qualcomm IPQ5424 global clock controllers Qualcomm QCS8300 global clock controllers, Renesas RTCA-3 realtime clocks, Marvell 88PM886 realtime clocks, and Amlogic A113L2(A4) and A113X2(A5) SoC realtime clocks. 
  * **GPIB** (general purpose interface bus) driver support was added to the staging tree; new drivers support National Instruments PCI/ISA compatible boards, Agilent 82357a/b USB dongles, Capital Equipment PCI-488 and Keithly KPCI-488 boards, Measurement Computing CPCI_GPIB, ISA-GPIB, ISA-GPIB/LC, PCI-GPIB/1M, PCI-GPIB/300K and PCMCIA-GPIB boards, fmhess FPGA-based devices, Raspberry Pi GPIO adapters, HP Agilent HP82335, HP27209 and HP27209 boards, numerous PC2 and PC2A-compatible adapters, TMS 9914 GPIB interfaces, and NEC 7210 GPIB interfaces. 
  * **GPIO and pin control** : Congatec GPIO and I2C controllers, FTDI MPSSE GPIO interfaces, Microchip FPGA GPIO controllers, T-Head TH1520 SoC pin controllers, Qualcomm IPQ5424, QCS8300, SM8750, SAR2130P, and QCS615 pin controllers, Canaan Kendryte K230 pin controllers, SpacemiT K1 SoC pin controllers, and Airoha EN7581 pin controllers. 
  * **Graphics** : TI TDP158 HDMI/TMDS bridges, Samsung S6E3HA8 DSI video mode panels, Samsung AMS639RQ08 and AMS581VF01 panels, Samsung AMS427AP24 panels with S6E88A0 controllers, Sharp Memory LCD panels, and ITE IT6263 LVDS/HDMI bridges. 
  * **I2C** : AMD ASF I2C controllers and Realtek RTL9300 I2C controllers. 
  * **Industrial I/O** : Bosch BMI270 6-Axis IMUs, Analog Devices AD8460 and AD3552R digital-to-analog converters, GE HealthCare PMC analog-to-digital converters, Vishay VEML6035 ambient light sensors, Analog Devices AD7625/AD7626 high-speed analog-to-digital converters, Bosch Sensor SMI240 IMUs, Analog Devices AD7779 analog-to-digital converters, Vishay VEML3235 ambient light sensors, and Allegro MicroSystems ALS31300 3-D linear Hall-effect sensors. 
  * **Input** : Corsair Void headsets. 
  * **Media** : Raspberry Pi RP1 Camera Front End (CFE) video capture interfaces. 
  * **Miscellaneous** : Realtek SPI-NAND flash controllers, Apple SoC SPI controllers, Congatec watchdog timers, UHS2-capable devices on SDHCI controllers, AMD 3D V-Cache performance optimizers, MediaTek dynamic voltage and frequency scaling resource collectors, Microchip LAN966x PCIe controllers, Marvell 88E6xxx network interface LEDs, RISC-V I/O memory-management units, T-head TH1520 mailboxes, Marvell OCTEON PCI hotplug controllers, STMicroelectronics STM32MP25 COMBOPHYs, NXP PTN3222 1-port eUSB2 to USB2 redrivers, TI OMAP TWL6030 battery charge interfaces, Renesas R-Car Gen4 E-FUSEs, and Qualcomm QCS8300, SAR2130P, and QCS615 interconnect providers. 
  * **Networking** : Hisilicon BMC gigabit Ethernet interfaces, Microchip Lan969x switches, NXP i.MX95 ENETC interfaces, Marvell RVU E-Switch switches, Realtek 8821AU/8811AU and 8812AU USB wireless network adapters, and KEBA CP500 LAN9252 interfaces. 
  * **Sound** : NeoFidelity NTP8918, NTP8835, and NTP8835C amplifiers, Realtek RT721 SDCA codecs, Everest Semi ES8323 codecs, NXP UDA1342 codecs, Cirrus Logic CS42L84 codecs, Awinic aw88081 amplifiers, and Iron Device SMA1307 audio amplifiers. 
  * **USB** : TI TUSB1046 Type-C crosspoint switches. 

#### Miscellaneous

  * [Trace events in Rust code](<https://lwn.net/Articles/992455/>) are now supported. 
  * Rust bindings for misc devices have been added, opening the door to the merging of a number of drivers in future development cycles. 
  * See [this merge message](<https://git.kernel.org/linus/b50ecc5aca4d>) for a summary of new features added to the `perf` tool. 
  * The build system has gained support for [optimization with AutoFDO and Propeller](<https://lwn.net/Articles/995397/>). 

#### Networking

  * Networking performance in many cases is constrained by contention on the RTNL lock — a problem that has been understood for years. The 6.13 kernel contains work turning the RTNL lock into a per-network-namespace lock as a way of reducing that contention in namespace-heavy workloads (running containers, for example). It is just one step in a long process, and the regression potential is considered to be significant, so the per-namespace behavior is disabled by default. The `DEBUG_NET_SMALL_RTNL` configuration variable can be used to turn it on. 
  * There is a new, more flexible API for the control of traffic shaping; [this merge message](<https://git.kernel.org/linus/bdb5d2481a53>) gives an overview of how it works. 
  * There is a new netlink API for the control of NAPI (polling) behavior in network drivers; [this merge message](<https://git.kernel.org/linus/5bedbfc16552>) provides an overview. 
  * The long-obsolete wireless extensions emulation within the kernel will no longer export a set of symbols to loadable modules. No in-tree drivers need that feature at this point; it is possible that out-of-tree drivers might break. 
  * There is a new mechanism allowing network drivers to suspend interrupts during busy-traffic periods; see [this merge message](<https://git.kernel.org/linus/80b6f094756f>) for more information. 

#### Virtualization and containers

  * The [IOMMUFD subsystem](<https://docs.kernel.org/userspace-api/iommufd.html>) has gained a number of new operations aimed at better support for virtual machines; see [this merge message](<https://git.kernel.org/linus/341d041daae5>) for an overview. 
  * The sound subsystem has a new pass-through mode for audio-related accelerators; [this commit](<https://git.kernel.org/linus/04177158cf98>) includes an overview document. 
  * The new virtual CPU-frequency driver allows CPU-frequency coordination between hosts and guests; see [this commit](<https://git.kernel.org/linus/4fd06a5358e0>) for a little more information. 

#### Internal kernel changes

  * There is a new memory allocator for Rust code that removes the dependency on the external `Alloc` module. Internal documentation for the associated `Allocator` trait can be found in [this commit](<https://git.kernel.org/linus/b7a084ba4fbb>). There are also kernel-specific implementations of `Box` and `Vec` that use this allocator; see [this commit](<https://git.kernel.org/linus/c8cfa8d0c0b1>) for some internal `Box` documentation and [this commit](<https://git.kernel.org/linus/2aac4cd7dae3>) for `Vec`. 
  * There were 231 exported symbols removed in 6.13, while 443 new exports were added; there are also 11 new kfuncs. See [this page](<https://lwn.net/Articles/1000371/>) for the full list. 

The 6.13 kernel development cycle now moves into the stabilization phase where, with luck, the bulk of the bugs that have inevitably been introduced will be fixed. The 6.13 release can be expected on January 19 or 26.
