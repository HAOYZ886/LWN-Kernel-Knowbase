---
title: The rest of the 6.10 merge window
url: https://lwn.net/Articles/974869/
date: "May 27, 2024"
category: "Releases-6.10"
author: "By Jonathan Corbet May 27, 2024"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 27, 2024

Linus Torvalds [released 6.10-rc1](<https://lwn.net/Articles/975144/>) and closed the 6.10 merge window on May 26\. By that time, 11,534 non-merge changesets had been pulled into the mainline for the next release; nearly 5,000 of those came in after "[The first half of the 6.10 merge window](<https://lwn.net/Articles/973687/>)" was written. While the latter half of the merge window tends to focus more on fixes, there was also a lot of new functionality that landed during this time. 

Significant changes merged since the first-half summary include: 

#### Architecture-specific

  * 32-Bit Arm systems can now be built with [Clang-based control-flow integrity](<https://lwn.net/Articles/898040/>). 
  * The PowerPC BPF JIT compiler now supports kfuncs. 
  * The RISC-V architecture has gained support for the Rust language. 

#### Core kernel

  * It is now possible to map tracing ring buffers directly into user space. See [this merge message](<https://git.kernel.org/linus/53683e408013>) and [this documentation commit](<https://git.kernel.org/linus/a1e0dd7ce38a>) for more information. 
  * An initial set of patches toward the eventual [consolidation of hugetlbfs](<https://lwn.net/Articles/974491/>) into the core memory-management subsystem has been merged; there should be no user-visible changes. 
  * The [ntsync subsystem](<https://lwn.net/Articles/961884/>), which provides a set of Windows NT synchronization primitives for Linux, has been merged. It is, however, marked as "broken" for this release and cannot yet be used for its intended purpose. 
  * After a significant amount of discussion and change, the [`mseal()` system call](<https://lwn.net/Articles/958438/>) was merged as one of the final features for this development cycle. `mseal()` allows a process to forbid future changes to portions of its address space; the initial application is in the Chrome browser, which will use it to strengthen its internal sandboxing. More information can be found in [this documentation commit](<https://git.kernel.org/linus/c010d0990082>). 

#### Filesystems and block I/O

  * There is a new netlink-based protocol for the control of the NFS server in the kernel; a new `nfsdctl` tool is said to be on its way into the nfs-utils package. 
  * The XFS filesystem continues to gain more online repair functionality. 
  * The filesystems in user space (FUSE) subsystem now supports integrity protection with [fs-verity](<https://docs.kernel.org/filesystems/fsverity.html>). 
  * The overlayfs filesystem is now able to create temporary files using the `O_TMPFILE` option. 

#### Hardware support

  * **Clock** : Sophgo CV1800 series SoCs clock controllers, STMicroelectronics stm32mp25x clocks, NXP i.MX95 BLK CTL clocks, and Epson RX8111 realtime clocks. 
  * **Media** : Broadcom BCM283x/BCM271x CSI-2 receivers and sixth-generation Intel image processing units. 
  * **Miscellaneous** : Acer Aspire 1 embedded controllers, Lenovo WMI camera buttons, ACPI Quickstart buttons, Lenovo Yoga Tablet 2 1380 fast chargers, Dell AIO UART backlight interfaces, MeeGoPad ANX7428 Type-C switches, Zhaoxin I2C interfaces, Lenovo SE10 watchdog timers, ARM MHUv3 mailbox controllers, Samsung HDMI PHYs, MediaTek 10GE SerDes XFI T-PHYs, and Rockchip USBDP COMBO PHY. 

#### Miscellaneous

  * The `perf` tool has, as usual, seen a lot of changes; details can be found in [this merge message](<https://git.kernel.org/linus/29c73fc794c8>). 

#### Networking

  * The new `IORING_CQE_F_SOCK_NONEMPTY` operation for io_uring can be used to determine whether there are more connection requests waiting on a socket. 

#### Security-related

  * The [Landlock](<https://docs.kernel.org/security/landlock.html>) security module is now able to apply policies to `ioctl()` calls; see [this documentation commit](<https://git.kernel.org/linus/a3746da89faf>) for a bit more information. 
  * The new `init_mlocked_on_free` boot option will cause any memory that is locked into RAM with [`mlock()`](<https://man7.org/linux/man-pages/man2/mlock.2.html>) to be zeroed if (and only if) it is freed without having been first unlocked with `munlock()`. The purpose is to protect memory that may be holding cryptographic keys from being exposed after an application crash. 

#### Internal kernel changes

  * Developers may be unaware of the [`no_printk()`](<https://elixir.bootlin.com/linux/latest/source/include/linux/printk.h#L122>) macro. Its job is to do nothing, but to preserve `printk()` statements in the code should somebody need to restore them for future debugging purposes. In prior kernels, `no_printk()` still contributed [indexing data](<https://lwn.net/Articles/857148/>) to the kernel image, even though it printed nothing; that has been fixed for 6.10. 
  * Some changes to how memory for executable code in the kernel is allocated have made it possible to enable ftrace and kprobes without the need to enable loadable-module support. 
  * Work items in [BH workqueues](<https://lwn.net/Articles/960041/>) can now be enabled and disabled; with this change, it should be possible to convert all tasklet users over to the new mechanism. 
  * The (sometimes controversial) [memory-allocation profiling](<https://lwn.net/Articles/974380/>) subsystem has been merged; this should help developers optimize memory use and track down memory leaks. See [this documentation commit](<https://git.kernel.org/linus/ebdf9ad4ca98>) for some more information. 
  * There are 371 more symbols exported to modules in 6.10, and 18 new kfuncs; see [this page](<https://lwn.net/Articles/975138/>) for the full list of changes. 

If this development cycle follows the usual timeline (and they all do anymore), then the final 6.10 release will happen on July 14 or 21\. Between now and then, though, there will be a need for a lot of testing and bug fixing. 

[Note that LWN subscribers can find more information about the contributions to 6.10-rc1 in [the LWN kernel-source database](</ksdb/>).]
