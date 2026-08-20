---
title: The rest of the 5.6 merge window
url: https://lwn.net/Articles/811230/
date: "February 10, 2020"
category: "Releases-5.6"
author: "By Jonathan Corbet February 10, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 10, 2020

Linus Torvalds [released the 5.6-rc1 prepatch](<https://lwn.net/Articles/812030/>) and closed the merge window on February 9; at that point, 10,780 non-merge changesets had been pulled into the mainline repository for 5.6. That is substantially less than recent development cycles (14,350 for 5.5, 14,619 for 5.4), but is similar to what was going on at this time last year (10,843 for 5.0-rc1 in January 2019). About 6,000 of those changes were pulled since [the first 5.6 merge-window article](<https://lwn.net/Articles/810780/>) was written; read on for what was included in those changes. 

#### Architecture-specific

  * The Intel [memory protection extension](<https://lwn.net/Articles/582712/>) feature, merged in 3.19, has been removed. This feature failed to take the world by storm, and support for it has been removed from the GCC compiler. 
  * The RISC-V architecture has gained support for the [KASAN](<https://www.kernel.org/doc/html/latest/dev-tools/kasan.html>) address sanitizer. 

#### Core kernel

  * The last users of the 32-bit `time_t` type have been fixed, and the associated types have been removed from the kernel. That means that, modulo a few loose ends, the kernel is now year-2038 ready. User-space work will still be required, and applications on 32-bit systems will need to use an updated C library. See [this merge commit](<https://git.kernel.org/linus/22b17db4ea05>) for more details. 
  * The io_uring subsystem has gained many more supported operations; see [this article](<https://lwn.net/Articles/810414/>) for details. 
  * The [`pidfd_getfd()` system call](<https://lwn.net/Articles/808997/>), which allows a process to extract an open file descriptor from another process, has been merged. 
  * The "bootconfig" mechanism allows the kernel to receive, at boot time, a file containing command-line options; its intended use appears to be for tasks like setting up boot-time kprobes. The `bootconfig` tool, found in the `tools` directory, can be used to add one of these files to an initramfs image. [This commit](<https://git.kernel.org/linus/7b9b816f4b9a>) contains documentation for this feature. There is also a new set of options to control boot-time tracing; see [this commit](<https://git.kernel.org/linus/47781947947a>) for documentation. 
  * [A change to how pipe waiting works](<https://git.kernel.org/linus/0ddad21d3e99>) promises to speed up parallel kernel builds — always a kernel developer's favorite workload — but can also trigger a bug with old versions of GNU Make. 

#### Filesystems and block I/O

  * The F2FS filesystem has gained compression support; see [this commit](<https://git.kernel.org/linus/4c8ff7095bef>) for some more information. 
  * The new `softreval` NFS mount option allows attribute revalidation to succeed from cached values should the server go down. From [the changelog](<https://git.kernel.org/linus/c74dfe97c104>): ""The use case for this option is for ensuring that we can still (slowly) traverse paths and use cached information even when the server is down."" 
  * NFS mounts over UDP are now disabled by default. 
  * NFS v4.2 server-to-server file copies are now supported. 
  * The [ZoneFS](<https://lwn.net/Articles/794364/>) filesystem, which facilitates low-level access to zoned block devices, has been merged. See [this commit](<https://git.kernel.org/linus/8dcc1a9d90c1>) for more information. 

#### Hardware support

  * **Clock** : Qualcomm MSM8998 multimedia clock controllers, Qualcomm SC7180 clock controllers, Qualcomm IPQ6018 global clock controllers, and NXP IMX8MP CCM clock controllers. 
  * **Graphics** : Boe Himax8279d panels, R-Car DU color-management modules, Xinpeng XPP055C272 panels, Leadtek LTK500HD1829 panels, and Sony ACX424AKP DSI command-mode panels. 
  * **Miscellaneous** : Rockchip image signal processing units, TI TPS61050/TPS61052 LED controllers, ROHM BD71828 Power Management ICs, Qualcomm WCD9340/WCD9341 codecs, SGI IOC3 PS/2 controllers, and Mediatek system companion processors. 
  * **PCI** : Broadcom Brcmstb PCIe host controllers and Intel Gateway PCIe host controllers. 

#### Memory management

  * There is a new `prctl()` operation called `PR_SET_IO_FLUSHER`. Its purpose is to flag a process that is part of the memory-reclaim I/O path and which should thus not be throttled when memory constraints get tight. Within the kernel, it works by setting the `PF_MEMALLOC_NOIO` and `PF_LESS_THROTTLE` flags in the task structure. 
  * The [dma-buf heaps subsystem](<https://lwn.net/Articles/792733/>), derived from the Android ION allocator, has been merged at last. 

#### Security-related

  * The `/dev/random` blocking pool [has been removed](<https://lwn.net/Articles/808575/>). This change makes `/dev/random` behave more like `/dev/urandom`, in particular preventing it from blocking on available entropy once the pool has been initialized. 

#### Virtualization and containers

  * Linux guests running under VirtualBox can now mount folders exported by the host. 

#### Internal kernel changes

  * The [`pin_user_pages()` API](<https://lwn.net/Articles/807108/>) has been merged, allowing the kernel to better track user-space pages that have been pinned into RAM. The full infrastructure for making use of this information is still to come, though. 
  * The build system now tests that all UAPI header files include `#ifdef` guards to prevent them from being included twice. 
  * There is a new API that can be used to generate synthetic trace events in the kernel; see [this commit](<https://git.kernel.org/linus/34ed63573b66>) for documentation. 

The kernel development community now has until late March or early April, when the final 5.6 release is expected, to find and fix the bugs introduced by all of that work.
