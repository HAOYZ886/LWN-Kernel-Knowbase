---
title: The rest of the 6.3 merge window
url: https://lwn.net/Articles/924384/
date: "March 6, 2023"
category: "Releases-6.3"
author: "By Jonathan Corbet March 6, 2023"
---

By **Jonathan Corbet**  
March 6, 2023

Linus Torvalds [released 6.3-rc1](<https://lwn.net/ml/linux-kernel/CAHk-=wgr1D8hb75Z+nn+4LXUnosp0HM+gP+YJEcEav1DgTC=Cw@mail.gmail.com/>) and closed the 6.3 merge window as expected on March 5\. By that time, 12,717 non-merge commits (and 848 merges) had found their way into the mainline kernel; nearly 7,000 of those commits came in after [the first-half merge-window summary](<https://lwn.net/Articles/923846/>) was written. The second half of the 6.3 merge window was thus a busy time, with quite a bit of new functionality landing in the mainline. 

Some of the most significant changes merged during this time are: 

#### Architecture-specific

  * RISC-V kernels can use the "ZBB" bit-manipulation extension, when present, to accelerate string operations. 
  * User-mode Linux (on x86-64 systems) now supports code written in Rust. 
  * LoongArch has gained support for kernel relocation, kernel address-space layout randomization, hardware breakpoints and watchpoints, and kprobes. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Core kernel

  * It is now possible to [create a non-executable memfd](<https://lwn.net/Articles/918106/>) and prevent execute permission from being enabled thereafter. 
  * The [DAMOS](<https://lwn.net/Articles/863753/>) memory-management interface has a new "filters" option; there is some documentation in [this commit](<https://git.kernel.org/linus/9b7f9322a530>). 
  * The new `PR_SET_MDWE` `prctl()` operation will cause any attempts to enable both write and execute permissions on memory in the target process to be denied; see [this commit message](<https://git.kernel.org/linus/b507808ebce2>) for more information. 

#### Filesystems and block I/O

  * The NFS filesystem (both the client and server sides) has gained support for AES-SHA2-based encryption. 
  * The filesystems in user space (FUSE) subsystem has a new [request extension](<https://git.kernel.org/linus/15d937d7ca8c>) mechanism that can be used to put additional information onto a request. The first use is to [add supplementary groups](<https://git.kernel.org/linus/8ed7cb3f279f>) to a filesystem request. 
  * Christian Brauner [has been added](<https://git.kernel.org/linus/3304f18bfcf5>) as a co-maintainer of the virtual filesystem (VFS) layer. 

#### Hardware support

  * **Clock** : MediaTek MT7981 basic clocks, Qualcomm SM6350 camera clock controllers, Qualcomm SM8550 TCSR clock controllers, Qualcomm SM8550 display clock controllers, Qualcomm SA8775 and QDU1000/QRU1000 global clock controllers, and NXP BBNSM realtime clocks. 
  * **Graphics** : Orise Technology ota5601a RGB/SPI panels, Visionox VTDR6130 1080x2400 AMOLED DSI panels, HIMAX HX8394 MIPI-DSI LCD panels, Intel "versatile processing unit" inference accelerators, and AUO A030JTN01 320x480 3.0" panels. Also, several ancient drivers (i810, mga, r128, savage, sis, tdfx, and via) have been removed; they were all marked as being obsolete in 2016. 
  * **Industrial I/O** : TI TMAG5273 low-power linear 3D Hall-effect sensors, TI LMP92064 and ADS7924 analog-to-digital converters, Maxim MAX5522 digital-to-analog converters, and NXP IMX93 analog-to-digital converters. 
  * **Media** : OmniVision OV8858 image sensors and Sony IMX296 and IMX415 sensors. 
  * **Miscellaneous** : Unisoc UFS SCSI host controllers, Intel MAX 10 board management controllers with PMCI, Kinetic KTZ8866 backlight controllers, Microchip 8250-based serial ports, Ultrasoc system memory buffers, CoreSight trace, profiling, and diagnostics monitors, Qualcomm SDM670 and SA8775P interconnects, Freescale i.MX6/7/8 PCIe controllers in endpoint mode, Richtek RT9471 and RT9467 battery chargers, Loongson LS2X I2C adapters, GXP I2C Interfaces, Xilinx DMA/Bridge subsystem DMA engines, StarFive JH71XX power-management units, Renesas RZ/V2M external power sequence controllers, Allwinner D1 PPU power domain controllers, MediaTek SoC regulator couplers, Qualcomm ramp controllers, and Qualcomm PMIC GLINK interfaces. 
  * **USB** : Renesas RZ/N1 USB function controllers, Renesas USB3.1 DRD controllers, Qualcomm SNPS eUSB2 PHYs, and Qualcomm SNPS eUSB2 repeaters. 
  * **Also** : there is a new `pata_parport` driver that can manage IDE drives connected by way of a parallel port. With this driver, much of the work is done by the ATA layer, and the old PARIDE drivers have been removed. In this new world, it is no longer possible to put both drives and a printer on the port; [this commit](<https://git.kernel.org/linus/246a1c4c6b7f>) has some more information. 

(Ask your parents if you're unfamiliar with parallel ports and IDE drives.) 

#### Miscellaneous

  * The [Nolibc](<https://lwn.net/Articles/920158/>) library has gained support for the s390 architecture and the Arm Thumb1 instruction set. 
  * The new `hwnoise` tool can measure timing jitter caused by hardware; [this commit](<https://git.kernel.org/linus/5dc3750e747f>) contains a man page describing its operation. 
  * The `perf` tool has seen a number of improvements; [this merge message](<https://git.kernel.org/linus/0df82189bc42>) has details. 
  * The kernel now can feature [a built-in Dhrystone benchmark test](<https://git.kernel.org/linus/d5528cc16893>). 

#### Virtualization and containers

  * KVM on x86 has gained the ability to support [Hyper-V extended hypercalls](<https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/tlfs/hypercall-interface#extended-hypercall-interface>). These calls are implemented by passing them through to user space on the host side. 
  * Also on x86, KVM has made it easier to restrict the performance-measurement-unit events that are available to the guest; [this commit](<https://git.kernel.org/linus/14329b825ffb>) has more information. 

#### Internal kernel changes

  * The internal `__GFP_ATOMIC` memory-allocation flag [has been removed](<https://lwn.net/Articles/920891/>). Almost nobody should notice the change. 
  * The (default) `V=0` make option has been removed. The `V=1` (verbose mode) and `V=2` (show the reason why a target was rebuilt) can now be selected together as `V=12`. 
  * There has been [a minor change](<https://git.kernel.org/linus/d4563201f33a>) to the developer's certificate of origin to make it clear that patches submitted using a nickname are acceptable. 
  * The internal representation of capabilities [has been changed](<https://git.kernel.org/linus/f122a08b197d>) to a simple `u64` value. The previous (array) representation had been designed to allow for more capabilities in the future, but as Torvalds noted in the changelog, ""the last thing we want to do is to extend the capability set any more"". 
  * Support for building with the Intel ICC compiler [has been removed](<https://git.kernel.org/linus/95207db8166a>). It has seemingly been broken for some time and nobody noticed. 

The next seven or eight weeks will be spent stabilizing this new code in preparation for the 6.3 release, which should happen on April 23 or 30.
