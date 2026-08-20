---
title: The conclusion of the 5.14 merge window
url: https://lwn.net/Articles/861695/
date: "July 12, 2021"
category: "Releases-5.14"
author: "By Jonathan Corbet July 12, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 12, 2021

The 5.14 merge window closed with the [5.14-rc1](<https://lwn.net/ml/linux-kernel/CAHk-=wjB5XBk4obhMPfrU3mnOakV9VgHAYOo-ZGJnB2X0DnBWA@mail.gmail.com/>) release on July 11\. By that time, some 12,981 non-merge changesets had been pulled into the mainline repository; nearly 8,000 of those arrived after [the first LWN 5.14 merge-window summary](<https://lwn.net/Articles/861248/>) was written. This merge window has thus seen fewer commits than its predecessor, which saw 14,231 changesets before the 5.13-rc1 release. That said, there is still a lot of interesting work that has found its way into the kernel this time around. 

Some of the more significant changes pulled in the second half of the 5.14 merge window include: 

#### Architecture-specific

  * The s390 architecture now supports booting kernels compressed with the [Zstandard](<https://facebook.github.io/zstd/>) (zstd) algorithm. 
  * The RISC-V architecture has gained support for transparent huge pages and support for the [KFENCE](<https://lwn.net/Articles/835542/>) memory-safety checker. 

#### Core kernel

  * The [control-group kill button](<https://lwn.net/Articles/855049/>) patch set has been merged; this feature allows the quick killing of all members of a control group by writing to the `cgroup.kill` virtual control file. 
  * There are two new options for the [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) system call: 

    * `MADV_POPULATE_READ` will fault in all pages within the indicated mapping for read access; the effect is the same as if the caller had manually looped through the range, accessing each page. No COW mappings will be broken by this operation. 
    * `MADV_POPULATE_WRITE`, instead, will fault in the pages for write access, breaking COW mappings if need be. 

The purpose of these operations, in either case, is to pay the cost of faulting in a range of memory immediately, allowing the application to run without page-fault-induced delays later on. They differ from the `MAP_POPULATE` option to [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) in that they can be invoked at any time rather than just when the memory is mapped. See [this commit](<https://git.kernel.org/linus/4ca9b3859dac>) for more information. 
  * The [`memfd_secret()`](<https://lwn.net/Articles/835342/>) system call has been merged. It creates a region of memory that is private to the caller; even the kernel cannot directly access it. See [this commit](<https://git.kernel.org/linus/1507f51255c9>) for a bit more information. 

#### Filesystems and block I/O

  * The ext4 filesystem has gained a new `ioctl()` command called `EXT4_IOC_CHECKPOINT`. This command forces all pending transactions out of the journal, and can also overwrite the space on the storage device used by the journal. This operation is part of [an effort](<https://lwn.net/Articles/854054/>) to prevent information leaks from filesystems. [This documentation commit](<https://git.kernel.org/linus/fd7b23be9205>) describes the new operation and its options. 
  * The [`quotactl_fd()` system call](<https://lwn.net/Articles/859679/>) has been added. This is the new form of `quotactl_path()` that was briefly added to 5.13 before being disabled as the result of API concerns. 
  * The F2FS filesystem can now compress files that are mapped with `mmap()`. There is also a new `nocompress_extension` mount option that disables compression for any file whose name matches the given extension(s). 

#### Hardware support

  * **Clock** : Qualcomm MDM9607 global clock controllers, Qualcomm SM6125 global clock controllers, Qualcomm SM8250 camera clock controllers, Renesas RZ/G2L family clock controllers, TI LMK04832 JESD204B-clock jitter cleaners, Ingenic JZ4760 clock controllers, and Huawei Hi3559A clocks. 
  * **Graphics** : ITE IT66121 HDMI bridges, ChromeOS EC ANX7688 bridges, Hyper-V synthetic video devices, and TI SN65DSI83 and SN65DSI84 DSI to LVDS bridges. There is also a new ["simpledrm" driver](<https://git.kernel.org/linus/11e8f5fd223b>) that provides a direct-rendering interface for simple framebuffer devices; there are also the inevitable 200,000+ lines of new amdgpu register definitions. 
  * **Industrial I/O** : TI TMP117 digital temperature sensors, TI TSC2046 analog-to-digital converters, TAOS TSL2591 ambient light sensors, Murata SCA3300 3-axis accelerometers, Sensirion SPS30 particulate matter sensors, STMicroelectronics LSM9DS0 inertial sensors, NXP FXLS8962AF/FXLS8964AF accelerometers, and Intel quadrature encoders. 
  * **Miscellaneous** : Microchip 48L640 EERAM chips, PrimeCell SMC PL351 and PL353 NAND controllers, SparkFun Qwiic joysticks, Richtek RT4831 backlight power controllers, Qualcomm PM8008 power-management ics, Xillybus generic FPGA interfaces for USB, Qualcomm SC7280 interconnects, generic CAN transceivers, Rockchip Innosilicon MIPI CSI PHYs, Allwinner SUN6I hardware spinlocks, and MStar MSC313e watchdogs. 
  * **Pin control** : Mediatek MT8365 pin controllers, Qualcomm SM6125 pin controllers, and IDT 79RC3243X GPIO controllers. 
  * **Sound** : NXP/Goodix TFA989X (TFA1) amplifiers, Rockchip RK817 audio codecs, and Qualcomm WCD9380/WCD9385 codecs. 
  * **Removals** : the "raw" driver, which provided unbuffered access to block devices under `/dev/raw`, has been removed. Applications needing this sort of access have long since moved to `O_DIRECT`, or at least that's the belief. 

#### Virtualization and containers

  * User-mode Linux now supports PCI drivers with a new PCI-over-virtio driver. 

#### Testing and tracing

  * The kunit self-test subsystem now supports running tests under QEMU; see [this documentation commit](<https://git.kernel.org/linus/12ca7a893d54>) for details. 
  * There are two new tracing mechanisms in 5.14. The "osnoise" tracer tracks application delays caused by kernel activity — interrupt handling and such. The "timerlat" tracer gives detailed information about delays in timer-based wakeups. The [osnoise](<https://git.kernel.org/linus/bce29ac9ce0b>) and [timerlat](<https://git.kernel.org/linus/a955d7eac177>) commits have more details and instructions on how to use these features. 

The 5.14 kernel is now in the stabilization phase. Unless something highly unusual happens, the final 5.14 release will happen on August 29 or September 5\. There is a lot of testing and bug-fixing to be done in the meantime.
