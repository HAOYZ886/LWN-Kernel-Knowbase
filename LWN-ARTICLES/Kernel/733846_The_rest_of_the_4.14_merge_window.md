---
title: The rest of the 4.14 merge window
url: https://lwn.net/Articles/733846/
date: "September 17, 2017"
category: "Releases-4.14"
author: "By Jonathan Corbet September 17, 2017"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 17, 2017

As is sometimes his way, Linus Torvalds [released 4.14-rc1](<https://lwn.net/Articles/733917/>) and closed the merge window one day earlier than some might have expected. By the time, though, 11,556 non-merge changesets had found their way into the mainline repository, so there is no shortage of material for this release. Around 3,500 of those changes were pulled after [the previous 4.14 merge-window summary](<https://lwn.net/Articles/733175/>); read on for an overview of what was in that last set. 

User-visible changes include: 

  * The disk quota subsystem has received a fair amount of performance work, increasing file-creation performance in ext4 (with quotas enabled) by a factor of about two. 

  * The CIFS filesystem can now read and write extended attributes when using the SMB3 protocol. 

  * The [heterogeneous memory management](<https://lwn.net/Articles/684916/>) patch set has finally been merged. HMM exists to enable the use of devices with their own memory-management units that can access main memory. See [this changelog](<https://git.kernel.org/linus/bffc33ec539699f045a9254144de3d4eace05f07>) for an overview of the current state of this work. 

  * The [namespaced file capabilities patch](<https://lwn.net/Articles/726816/>) has been merged. This allows file capabilities to be used within user namespaces. The scheme has been simplified somewhat, though, so that only a single `security.capability` extended attribute can exist for any given file. 

  * The kernel has gained support for the [zstd](<https://github.com/facebook/zstd>) compression algorithm; it claims better performance and a better compression ratio. See [this changelog](<https://git.kernel.org/linus/e7cdb60fd28b252f1c15a0e50f79a01906124915>) for an overview. 

  * The new `IOCB_NOWAIT` flag can be used to specify that asynchronous buffered block I/O operations should be as close to non-blocking as possible. Without this flag (or in current kernels) these operations can block on memory-management operations, for example. 

  * The entire `firmware` subtree has been removed from the kernel repository. The firmware that everybody actually uses has been maintained [in its own repository](<https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/>) for some years now, so these files were unused and unneeded. 

  * New hardware support includes: Rockchip RK805 power-management chips, STMicroelectronics STM32 low-power timers, ROHM BD9571MWV power-management chips, TI TPS68470 power-management chips, Altera/Intel mSGDMA engines, Mediatek MT2712 and MT7622 PCI host controllers, Spreadtrum SC9860 I2C controllers, NXP i.MX6/7 remote processors, Maxim MAX17211/MAX17215 fuel gauges, Intel PCH/PCU SPI controllers, Qualcomm "B" family I/O memory-management units, Synopsys HSDK reset and clock controllers, ZTE ZX pulse-width modulators, Socionext UniPhier thermal monitoring units, Realtek RTD129x realtime clocks, Synopsys AXS10X SDP Generic PLL clocks, Renesas R-Car USB 2.0 clocks, Allwinner R40, A10, and A20 clock controllers, STMicroelectronics STM32H7 reset and clock controllers, Altera Soft IP I2C interfaces, STMicroelectronics STM32F7 I2C controllers, Lantiq XWAY SoC RCU reset controllers, Lantiq XWAY SoC RCU USB 1.1/2.0 PHYs, and PWM-controlled vibrators. 

Changes visible to kernel developers include: 

  * The [structure-layout randomization plugin](<https://lwn.net/Articles/722293/>) for GCC will now automatically rearrange the members of structures that consist entirely of function pointers. 

  * The build system has long used [gperf](<https://www.gnu.org/software/gperf/>) to generate hashes. As of 4.14, though, that is no longer true. The gperf 3.1 broke that generation and, rather than fix it in a version-independent way, Linus simply [ripped it out](<https://git.kernel.org/linus/5969d1bb3082b41eba8fd2c826559abe38ccb6df>). 

  * The new `memset16()`, `memset32()`, and `memset64()` functions can be used to set a range of memory to an integer value. 

  * `dma_alloc_noncoherent()` [has been removed](<https://lwn.net/Articles/732107/>); `dma_alloc_attrs()` should be used instead. 

Now it is just a matter of finding and fixing the various bugs that were inevitably introduced with all of those changes. If that process goes well and follows the usual schedule, the final 4.14 release can be expected on November 5 or 12.
