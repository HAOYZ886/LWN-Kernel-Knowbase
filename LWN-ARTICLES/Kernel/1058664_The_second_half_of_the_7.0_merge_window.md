---
title: The second half of the 7.0 merge window
url: https://lwn.net/Articles/1058664/
date: "February 23, 2026"
category: "Releases-7.0"
author: "By Daroc Alden February 23, 2026"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Daroc Alden**  
February 23, 2026

The 7.0 merge window [ closed](<https://lwn.net/ml/all/CAHk-%3DwiiRA_XxoF96Q_1n4BadBGJLRkHarHG92u3aTc%2B1ZMeGQ%40mail.gmail.com/>) on February 22 with 11,588 non-merge commits total, 3,893 of which came in after [the article covering the first half of the merge window](<https://lwn.net/Articles/1057769/>). The changes in the second half were weighted toward bug fixes over new features, which is usual. There were still a handful of surprises, however, including 89 separate tiny code-cleanup changes from different people for the rtl8723bs driver, a number that [surprised](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a5f22b9b1397>) Greg Kroah-Hartman. It's unusual for a WiFi-chip driver to receive that much attention, especially a staging driver that is not yet ready for general use. 

The most important changes included in this release were: 

#### Architecture-specific

  * The kernel has gained [ support for the RISC-V Zicfiss and Zicfilp extensions](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=79dd4f2f40d0>), which are used to provide hardware-assisted control-flow-integrity tracking in user space. 
  * LoongArch now supports [128-bit atomic compare-and-exchange instructions](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f0e4b1b6e295>) and [symmetric multithreading (SMT) hotplugging](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=009ee0c96416>). The latter can be controlled via sysfs configuration or boot parameters. 

#### Core kernel

  * The kernel's [ zram subsystem](<https://docs.kernel.org/admin-guide/blockdev/zram.html>) provides a compressed, in-memory block device that can optionally move data to a physical disk when memory fills up. Previously, the kernel would have to decompress the pages before writing them to the physical device. Now, [page writeback can directly write zram-compressed data](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d38fab605c66>). 
  * The swap subsystem has changed to [ use a simplified swap table](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f1879e8a0c60>), which has allowed for considerable code cleanups; see [this article](<https://lwn.net/Articles/1056405/>) and its [successor](<https://lwn.net/Articles/1057102/>) for more information. 
  * The `softlockup_panic` sysctl can now be used to [ set a desired timeout](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e700f5d15607>) after which the kernel should panic when it experiences a soft lockup, in multiples of 20 seconds. Previously, it was only an on-or-off setting. 

#### Filesystems and block I/O

  * [Laptop mode](<https://lwn.net/Articles/65437/>), a feature from the 2.6 kernel that reduces the number of times a hard disk needs to spin up or down to save on battery power, [has been removed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=64dd89ae01f2>). Spinning hard drives have largely been supplanted by solid-state drives, and ""the juice doesn't appear worth the squeeze anymore."" 
  * The conversion of filesystems to use large folios continues apace, with [ F2FS being the next to receive support](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3e48a11675c5>). [ F2FS](<https://www.kernel.org/doc/html/latest/filesystems/f2fs.html>) is a filesystem specialized for NAND flash devices. 
  * The ntfs3 filesystem has received [ a flurry of work](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=75a452d31ba6>) after a long period of torpor. At the same time, [work to replace the implementation entirely](<https://lwn.net/Articles/1055062>) has been circulated on the mailing list. Despite [ approval](<https://lwn.net/ml/all/20260204094210.GA31939%40lst.de/>) from reviewer Christoph Hellwig, the new NTFS implementation was not merged this cycle. 
  * NFSD, the network file system daemon, has added a [dynamically adjustable thread-pool](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1c87a0c39a86>) that scales up and down depending on demand. The behavior can be controlled via NFSD's netlink interface. 
  * NFS protocol version 4.1 is [the newly chosen default](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7537db24806f>), although users can still select older versions in their build configuration. 
  * NFS will [now refuse to export special kernel filesystems](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b3c78bc53630>) such as pidfs and nsfs, since these filesystems require special handling of permissions that NFSD does not do. 
  * Experimental support for [ POSIX ACLs](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=feb8a46b14d9>) has been added to NFSD, following a [ draft proposal](<https://www.ietf.org/archive/id/draft-ietf-nfsv4-posix-acls-00.txt>) to add them to NFS v4.2. 

#### Hardware support

  * **Clock** : Qualcomm Kaanapali clock controllers, Qualcomm SM8750 camera clock controllers, Qualcomm MSM8940 and SDM439 global clock controllers, Google GS101 DPU clock controllers, SpacemiT K3 clock controllers, Amlogic t7 clock controllers, Aspeed AST2700 clock controllers, and Loongson-2K0300 real-time clocks. 
  * **GPIO and pin control** : Spacemit K3 pin controllers, Atmel AT91 PIO4 SAMA7D65 pin controllers, Exynos9610 pin controllers, Qualcomm Mahua TLMM pin controllers, Microchip Polarfire MSSIO pin controllers, and Ocelot LAN965XF pin controllers. 
  * **Graphics** : Mediatek MT8188 HDMI PHYs, Mediatek Dimensity 6300 and 9200 DMA controllers, Qualcomm Kaanapali and Glymur GPI DMA engines, Synopsis DW AXI Agilex5 DMA devices, Atmel microchip lan9691-dma devices, and Tegra ADMA tegra264 devices. 
  * **Industrial I/O** : AD18113 amplifiers, AD4060 and AD4052 analog-to-digital converters (ADCs), AD4134 24-bit 4-channel simultaneous sampling ADCs, ADAQ767-1 ADCs, ADAQ7768-1 ADCs, ADAQ7769-1 ADCs, Honeywell board-mount pressure and temperature sensors, mmc5633 I2C/I3C magnetometers, Microchip MCP47F(E/V)B(0/1/2)(1|2|4|8) buffered-voltage-output digital-to-analog converters (DACs), s32g2 and s32g3 platform ADCs, ADS1018 and ADS1118 SPI ADCs, and ADS131M(02/03/04/06/08)24-bit simultaneous sampling ADCs. 
  * **Input** : FocalTech FT8112 touchscreen chips, FocalTech FT3518 touchscreen chips, and TWL603x power buttons. 
  * **Media** : MediaTek MT8196 video companion processors and external memory interfaces. 
  * **Miscellaneous** : Foresee F35SQB002G chips, TI LP5812 4x3 matrix RGB LED drivers, Osram AS3668 4-channel I2C LED controllers, Qualcomm Interconnect trace network on chip (TNOC) blocks, and DiamondRapids non-transparent PCI bridges. 
  * **Networking** : Glymur bandwidth monitors, Glymur PCIe Gen4 2-lane PCIe PHYs, SC8280xp QMP UFS PHYs, Kaanapali PCIe PHYs, and TI TCAN1046 PHYs. 
  * **Power** : ROHM BD72720 power supplies, Rockchip RK801 power management integrated circuits (PMICs), ROHM BD73900 PMICs, Delta Networks TN48M switch complex programmable logic devices (CPLDs), sama7d65 XLCD controllers, and Congatec Board Controller backlights. 
  * **USB** : AST2700 SOCs, USB UNI PHY and SMB2370 eUSB2 repeaters, QCS615 QMP USB3+DP PHYs, SpacemiT PCIe/combo PHY and K1 USB2 PHYs, Renesas RZ/V2H(P) and RZ/V2N USB3 devices, Google Tensor SoC USB PHYs, and Apple Type-C PHYs. 

#### Miscellaneous

  * The [ timerlat](<https://docs.kernel.org/tools/rtla/rtla-timerlat.html>) tool, which helps measure scheduler latency, has gained [an option](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f967d1eca7d0>) for running BPF programs when an operation exceeds a given latency threshold. 
  * The tracing subsystem has also gained [an option for more human-readable bitmasks](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2cddfc2e8fc7>) (displaying the bits as a list instead of raw hex) and a way to audit active [filters](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=729757b96a66>) and [triggers](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6a80838814ee>) in tracefs. 
  * A new build-time configuration option [allows people to replace the Tux boot logo](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dfa6ce636cb8>) with an image of their choosing. 
  * A new [ "`perf sched stats`" command](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=800af362d689>) can capture scheduler statistics (schedstat counters) and report on them. It can also produce diffs between previous reports for ease of comparison. 

#### Networking

  * The AccECN congestion-notification protocol has been [ enabled for general use](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=667539f6dce27aa7db0a711375f94e14e714a698>); see [this article](<https://lwn.net/Articles/1058666/>) for details. 
  * The [ cake queue-management system](<https://www.bufferbloat.net/projects/codel/wiki/Cake/>) has been [ extended to support multiple queues](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ff420c568b4c>). This lets cake's rate-shaper scale across multiple CPUs. 
  * [ VSOCK sockets](<https://www.man7.org/linux/man-pages/man7/vsock.7.html>), which are used to communicate with virtual machines, have gained [ support for network namespaces](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eafb64f40ca4>). 
  * Work on [802.11bn](<https://en.wikipedia.org/wiki/IEEE_802.11bn>) (perhaps better known as "Ultra High Reliability" WiFi or "WiFi 8") has [ started](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a1085114715ee9980405d6856276c5e88339cee7>), although there are many details to be finalized before it will be fully supported. 

#### Virtualization and containers

  * KVM has added a mask to [correctly report CPUCFG bits](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=de0c51370b7d>) on LoongArch, which provides LoongArch guests with the correct information about the CPU's capabilities. 
  * Some AMD CPUs support Enhanced Return Address Predictor Security (ERAPS) — a feature that removes the need for some security-related flushes of CPU state when a guest exits back to the host operating system. KVM added [support for using ERAPS](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=db5e82496492>), and for advertising that support to guests. 
  * There's [a new user-space control](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6517dfbcc918>) to configure KVM's end of interrupt (EOI) broadcast suppression (which prevents an EOI signal from being sent to all interrupt controllers in a system, making interrupts more efficient). Previously, KVM erroneously advertised support for EOI broadcast suppression, even though it wasn't fully implemented. Unfortunately, the flaw persisted long enough that some user-space applications came to depend on the behavior, so now that the feature has been implemented correctly, user-space programs will have to opt in when configuring KVM virtual machines. 
  * Guests can now request [full ownership of performance monitoring unit (PMU) hardware](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bf2c3138ae36>), which provides more accurate profiling and monitoring than the existing emulated PMU. 
  * The kernel's Hyper-V driver has added [a debugfs interface](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ff225ba9ad71>) to view various statistics about the Microsoft hypervisor. 

#### Internal kernel changes

  * The kernel has been [almost entirely switched over to `kmalloc_obj()`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8934827db540>) through the use of Coccinelle. Allocations of structure types and union types have all been converted, but allocations of scalar types, which need manual checking, have been left alone. Linus Torvalds followed up with a [handful](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fa5c82f4d2bb>) of [ fixes](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e19e1b480ac7>) for the [problems](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=32a92f8c8932>) that inevitably crop up after this kind of large change. The new interface allocates memory based on the size of the provided type, which should mean fewer mistakes with manual size calculations. Where one would previously write one of these: 

```
ptr = kmalloc(sizeof(*ptr), gfp);
            ptr = kmalloc(sizeof(struct some_obj_name), gfp);
            ptr = kzalloc(sizeof(*ptr), gfp);
            ptr = kmalloc_array(count, sizeof(*ptr), gfp);
            ptr = kcalloc(count, sizeof(*ptr), gfp);
            ptr = kmalloc(struct_size(ptr, flex_member, count), gfp);
```

One can now write: 

```
ptr = kmalloc_obj(*ptr, gfp);
            ptr = kmalloc_obj(*ptr, gfp);
            ptr = kzalloc_obj(*ptr, gfp);
            ptr = kmalloc_objs(*ptr, count, gfp);
            ptr = kzalloc_objs(*ptr, count, gfp);
            ptr = kmalloc_flex(*ptr, flex_member, count, gfp);
```

`GFP_KERNEL` is also now the default, and can be left out if that is the only memory allocation flag that one wishes to set. 

The second-half of the merge window is often quieter, and this one was no exception. There were a number of debugging features added, however, which is always nice to see. At this point, the kernel will go through the usual seven-or-eight release candidates as people chase down bugs introduced in this merge window. The final v7.0 kernel should be expected around April 12 or 19.
