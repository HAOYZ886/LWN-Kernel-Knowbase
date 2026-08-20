---
title: The end of the 6.18 merge window
url: https://lwn.net/Articles/1041004/
date: "October 14, 2025"
category: "Releases-6.18"
author: "By Daroc Alden October 14, 2025"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Daroc Alden**  
October 14, 2025

The 6.18 merge window has come to an end, bringing with it a total of 11,974 non-merge commits, 3,499 of which came in after LWN's [first-half summary](<https://lwn.net/Articles/1040203>). The total is a little higher than the 6.17 merge window, which saw 11,404 non-merge commits. There are once again a good number of changes and new features included in this release. 

The most important changes included in the second part of the 6.18 merge window were: 

#### Architecture-specific

  * The kernel [ now understands](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8f7729552582>) the RISC-V IO Mapping Table (RIMT) that provides information on some kinds of IOMMU via ACPI. 
  * The [ RISC-V RPMI platform-management communication interface](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4752b0cfbc37>) and [ MPXY SBI firmware shared-memory communication system](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bf3022a4eb11>) are both now supported. 
  * Static keys, which depend on having an implementation of the `jump_label()` helper, [ are now supported on openrisc](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8c30b0018f9d>). 
  * Trust domain extensions (TDX) on x86 and kexec [ can now be used at the same time](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=50ac57c3b156>); previously, the encryption hardware underlying TDX would overwrite cachelines belonging to the new kernel. 
  * User-mode Linux (UML) [now supports sparse interrupts](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=35fae10aaf08>) to better support the interrupt KUnit tests. 

#### Core kernel

  * The [ Rust Binder driver](<https://lwn.net/Articles/953116/>) has [ finally been merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eafedbc7c050>), after many necessary supporting APIs were added. The Binder maintainers say that the C implementation will remain in the kernel for several more releases while they verify that the Rust implementation precisely matches the existing user-space interface. Once that happens, it will be a true point of no return for Rust in the kernel, given that Binder is both essential to Android and receiving constant development. 

#### Filesystems and block I/O

  * Calls to [ `umount()`](<https://www.man7.org/linux/man-pages/man2/umount.2.html>) should [ no longer provoke quadratic behavior](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=75db7fd99075>) when a mount contains many other mounts. 
  * FUSE filesystems can [ now perform direct copies](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7a37f55af7af>) of ranges larger than 32 bits in size. This lets such filesystems support [ `copy_file_range()`](<https://man7.org/linux/man-pages/man2/copy_file_range.2.html>)
  * The ext4 filesystem now has [ 32-bit reserved user and group IDs](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=12c84dd4d308>) and [ a new `ioctl()` interface](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=04a91570ac67>) to get and set superblock parameters. Some legacy ext3 kernel-configuration options, which remained after the old ext3 filesystem was subsumed by ext4, [ have now been fully removed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d6ace46c82fd>). 
  * A new device-mapper target ([dm-pcache](<https://origin.kernel.org/doc/html/latest/admin-guide/device-mapper/dm-pcache.html>)) can [use persistent memory](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1d57628ff95b>) as a cache for slower block devices. 
  * The f2fs filesystem [ now has](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=632f0b6c3e32>) a lookup_mode mount option to control the performance characteristics of accessing case-folded directories. The chosen mode can be [ read](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1bd119da0b93>) from a new sysfs entry. 
  * Overlayfs now [ supports](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cf06d791f840>) case-folding as well, although not on a per-directory basis. Each overlayfs layer must be entirely case-folded or not case-folded. 

#### Hardware support

  * **Clock** : Allwinner A523/T527 MCU clock control units, Qualcomm ipq5424 APSS clock controllers, Qualcomm Glymur SoC TCSR, global, and display clock controllers, Qualcomm MSM8937 SoC clocks, STMicroelectronics STM32MP21 clocks, MediaTek MT8195 vencsys clocks, MediaTek MT8196 vencsys, vdecsys, ufssys, pextpsys, I2C, mcu, mdpsys, mfg, and disp0 clock controllers, and SpacemiT P1 real-time clocks. 
  * **Graphics** : Qualcomm QCS8300 eDP PHYs. 
  * **Industrial I/O** : Vishay VEML6046X00 high accuracy RGBIR color sensors, Intel Bay Trail/Cherry Trail Dollar Cove TI PMIC ADC devices, Infineon TLV493D 3D magnetometers, Analog Devices ADE9000 multiphase power monitors, Marvell 88PM886 PMIC analog-digital converters, and Rohm BD79112 analog-digital converters. 
  * **Input** : Awinic AW86927 haptic chips, Hynitron CST816x series controllers, and Himax HX852x(ES) touchscreen controllers. 
  * **Miscellaneous** : ECB and CBC modes of Texas Instruments DTHE V2 hardware AES engines, Xilinx Versal random-number generators, ThinkPad T14s Gen6 Snapdragon embedded controllers, Acer A1-840 tablets, Loongson-2K100 and 2K0500 NAND controllers, FudanMicro FM25S01A NAND controllers, Realtek RTl93xx switch SoC ECC controllers, STMicroelectronics M2LRxx I2C devices, ADLink PCI-7250, LPCI-7250, and LPCIe-7250 PCI/PCIe boards, Airoha AN8855 switch electronic fuses, SpacemiT K1 PDMA controllers, Sophgo SG2042 SoC PCIe controllers, STMicroelectronics STM32MP25 PCIe controllers, and MediaTek MT8196 SoC GPUEB IPI mailboxes. 
  * **Networking** : Pensando Ethernet RDMA devices and MediaTek PCIe 5G HP DRMR-H01 networking cards. 
  * **Sound** : DualSense Playstation controller audio jacks. 
  * **USB** : Intel USBIO USB IO-expanders, Renesas RZ/G3E SoC USB host controllers, Maxim MAX14526 MUIC devices, Qualcomm PMIV0104 eUSB2 repeaters, and Sophgo CV18XX/SG200X USB PHYs. 

#### Miscellaneous

  * Human interface device (HID) drivers can now handle [ haptic touchpads](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b44779d44f71>). 
  * Kernel hand-over can [ now preserve](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a667300bd53f>) `vmalloc()` allocations. 

#### Security-related

  * Following feedback suggesting that it is broken, the new HMAC-encrypted-transaction support on the trusted-platform module (TPM) bus [ has been disabled](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4bddf4587c13>) by default. 

#### Virtualization and containers

  * x86 hosts can now enable [SEV-SNP CipherText Hiding](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=10ef74c06bb1>), which prevents unauthorized CPU accesses from reading the ciphertext of secure nested paging (SNP) guests' private memory. 
  * KVM supports [ virtualizing control-flow enforcement technology](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=12abeb81c873>) (CET) on Intel (in the form of shadow stacks and indirect branch tracking) and AMD (in the form of shadow stacks) chips. While shadow stacks and indirect branch tracking can theoretically be enabled separately, in practice that has performance problems, so KVM will enable both if either one is requested. 

#### Internal kernel changes

  * The build documentation now [ clarifies](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bc20c56e98e0>) that Python is a required dependency for many kernel configurations. 
  * The DMA-mapping API [ continues the migration to physical addresses](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a498d59c469b>) as the primary interface, rather than page pointers and offsets. The [patch set](<https://lwn.net/ml/all/cover.1757423202.git.leonro@nvidia.com>) gives the justification, and [ another commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e1d69da24fb8>) in this series shows a sample conversion. 
  * Relatedly, Greg Kroah-Hartman has [ said](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6093a688a07da07808f0122f9aa2a3eed250d853>) that all of the pieces needed to implement a typical USB driver in Rust are now in place — although, with only [ a sample driver](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cc80dbb73b5d>) using the abstractions, the bindings themselves are [ disabled](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c584a1c7c8a1>) in the build until someone submits an actual, working USB driver and any remaining problems are fixed. 
  * The work to support memory descriptors continues; the kernel now has [ a new `memdesc_flags_t` structure](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=53fbef56e07d>) to separate out the flags from slab and folio structures. 
  * The work on [huge zero folios](<https://lwn.net/Articles/1033058/>) has been merged. 
  * The [`prctl()` interface](<https://lwn.net/Articles/1032199/>) for transparent huge pages has [ also made it in](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9dc21bbd62ed>) to this release. 
  * The `perf` utility now [ handles](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=945f50036169>) "N" (debugging) symbols produced by the Rust compiler. It also has a new "X" modifier [ that prevents events from being grouped automatically](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=035c17893082>). 
  * There were 190 exported symbols removed during this merge window, and 353 added; see [this page](<https://lwn.net/Articles/1041911/>) for the full list. 

  * There were also ten new kfuncs added: `bpf_dynptr_from_skb_meta()`, `bpf_kfunc_multi_st_ops_test_1()`, `bpf_kfunc_ret_rcu_test()`, `bpf_kfunc_ret_rcu_test_nostruct()`, `bpf_strcasecmp()`, `bpf_task_work_schedule_resume()`, `bpf_task_work_schedule_signal()`, `bpf_xdp_pull_data()`, `scx_bpf_cpu_curr()`, and scx_bpf_locked_rq(). 

Linus Torvalds seemed [pleased](<https://lwn.net/Articles/1041657/>) that this merge window didn't introduce any problems on his test machines; other than that happy occurrence, the 6.18-rc1 release was fairly typical. 6.18 is expected to become the next long-term support release of the kernel. If the release-candidate process takes the same amount of time as previous releases, 6.18 proper will likely be released around the end of November.
