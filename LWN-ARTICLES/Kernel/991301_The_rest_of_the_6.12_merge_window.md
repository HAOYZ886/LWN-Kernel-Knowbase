---
title: The rest of the 6.12 merge window
url: https://lwn.net/Articles/991301/
date: "September 30, 2024"
category: "Releases-6.12"
author: "By Jonathan Corbet September 30, 2024"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 30, 2024

Linus Torvalds [released 6.12-rc1](<https://lwn.net/ml/all/CAHk-=wiwVOCZsC6a4dLdb1UjL2fS_CnLNjrPL0XGFbDd9C26Cg@mail.gmail.com/>) and closed the 6.12 merge window on September 29; at that point, 11,260 non-merge change sets had been pulled into the mainline for the 6.12 release. That is the lowest number of merge-window changes since 5.17-rc1 in January 2022, which brought in 11,068 changesets. Nonetheless, 6.12 brings a number of interesting changes, many of which were included in the roughly 4,500 changes merged since [the summary of the first half of the 6.12 merge window](<https://lwn.net/Articles/990750/>) was written. 

Significant changes entering the kernel in the second half of the merge window include: 

#### Core kernel

  * Several features of the version-1 control-group interface, including TCP accounting, the v1 soft limits, out-of-memory control, and pressure level, have been deprecated. As was [discussed](<https://lwn.net/Articles/974575/>) earlier this year, the first step is to emit warnings to see how many users (if any) are affected at this point. 
  * After a year of [controversy](<https://lwn.net/Articles/972710/>), the [extensible scheduler class](<https://lwn.net/Articles/922405/>) has been merged. This feature allows the writing of CPU schedulers as a set of BPF programs; it has already [resulted](<https://lwn.net/Articles/991205/>) in a number of interesting, specialized schedulers being written. [This commit](<https://git.kernel.org/linus/fa48e8d2c7b5>) contains some introductory documentation. 
  * It is now possible to set up the tracing ring buffer in a way that allows it to persist across a reboot, perhaps saving information that would otherwise be lost in a crash. See [this changelog](<https://git.kernel.org/linus/af9c191ac2a0>) for some details. 

#### Filesystems and block I/O

  * The virtual filesystem layer has gained the ability to work with storage devices that have a block size larger than the system's page size. Getting to this point has required a multi-year effort, the last stages of which were [discussed](<https://lwn.net/Articles/974219/>) at the Linux Storage, Filesystem, Memory-Management, and BPF Summit earlier this year. At this point, only the XFS filesystem is able to take advantage of this capability. 
  * The filesystems in user space (FUSE) subsystem had gained support for [ID-mapped mounts](<https://lwn.net/Articles/896255/>). 
  * The NFS filesystem now supports the "LOCALIO" protocol, which allows the client and server to determine whether they are running on the same host; there are some significant performance improvements that can be had in that case. See [this documentation commit](<https://git.kernel.org/linus/92945bd81ca4>) and [this one](<https://git.kernel.org/linus/f7128262b152>) for more information. 

#### Hardware support

  * **Clock** : Rockchip RK3576 clock controllers, Qualcomm SM8150 camera clock controllers, Qualcomm SM4450 display clock controllers, Qualcomm SM4450 camera clock controllers, Qualcomm SM4450 graphics clock controllers, and DFRobot SD2405AL I2C realtime clocks. 
  * **GPIO and pin control** : Mobileye EyeQ5 pin controllers and Sophgo CV1800B, CV1812H, SG2000, and SG2002 SoC pin controllers. 
  * **Industrial I/O** : TYHX HX9023S SAR sensors, Analog Devices ADXL380 3-axis accelerometers, Analog Devices LTC2664 and LTC2672 digital-to-analog converters, Analog Devices AD4695 and AD4000 analog-to-digital converters, ROHM BH1745 color sensors, Microchip Technology PAC1921 power/current monitors, Sensirion SDP500 differential pressure sensors, ScioSense ENS210 temperature and humidity sensors, Sophgo CV1800B analog-to-digital converters, and Awinic AW96103/AW96105 proximity sensors. 
  * **Input** : PixArt PS/2 touchpads. 
  * **Media** : Extron DA HD 4K Plus CEC HDMI distribution amplifiers. 
  * **Miscellaneous** : native PCIe enclosure LEDs, Renesas RZ/V2H(P) watchdog timers, Nuvoton MA35 USB2.0 PHYs, AMD Queue-based DMA engines, Loongson1 APB DMA controllers, NXP LPC32xx DMA multiplexers, BlinkM I2C RGB LED controllers, Theobroma Systems Mule I2C device multiplexers, KEBA I2C controllers, TI K3 M4 remote processors, Qualcomm MSM8976 and MSM8937 interconnects, and ON Semiconductor LC824206XA microUSB switches. 

#### Miscellaneous

  * The user-space `perf` tool has gained a number of new features; see [this commit](<https://git.kernel.org/linus/891e8abed532>) for a summary. 
  * Rust code now works with a number of kernel development and security features, including the KASAN sanitizer, Spectre mitigations, and kernel control-flow integrity. Most GCC plugins can now be used with Rust as well, with the only exception being [structure-layout randomization](<https://lwn.net/Articles/722293/>). 
  * There are new Rust modules for doubly linked lists and red-black trees. 

#### Security-related

  * The Landlock security module now supports the concept of "scoping" for Unix-domain sockets, allowing the administrator to limit communications between sandboxes. [This documentation commit](<https://git.kernel.org/linus/dba40c770038>) has more information. The scoping concept also applies to signals; see [this commit](<https://git.kernel.org/linus/1ca980815e1f>) for (a little) more information. 
  * The Tomoyo security module can now be built as a loadable kernel module. This is a bit of a surprising change, since making security modules loadable has been resisted by the relevant maintainers for years. The [main commit](<https://git.kernel.org/linus/8b985bbfabbe>) also includes a novel mechanism for bypassing the normal restrictions on access to (some) non-exported symbols by loadable modules. [**Update** : this change was subsequently reverted.] 

#### Internal kernel changes

  * The new `xdrgen` tool exists for ""translating XDR specifications into XDR encoder and decoder functions written in the Linux kernel's C coding style"". See [this commit](<https://git.kernel.org/linus/4b132aacb076>) for more information. 
  * Torvalds added [a new mechanism for user-space address masking](<https://git.kernel.org/linus/2865baf54077>) that is intended to regain some of the performance that was lost to Spectre mitigations. It only works on x86 systems for now; it works through the application of a data-dependent mask that prevents speculation, eliminating the need to use more expensive barriers. 

The 6.12 release is feature-complete at this point; all that remains is to find and fix all of the bugs that were inevitably added as well. That work will be carried out through the upcoming series of -rc releases, culminating in the 6.12 release in the second half of November.
