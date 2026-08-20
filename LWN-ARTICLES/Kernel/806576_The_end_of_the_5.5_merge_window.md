---
title: The end of the 5.5 merge window
url: https://lwn.net/Articles/806576/
date: "December 9, 2019"
category: "Releases-5.5"
author: "By Jonathan Corbet December 9, 2019"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 9, 2019

By the end of the merge window, 12,632 non-merge changesets had been pulled into the mainline repository for the 5.5 release. This is thus a busy development cycle — just like the cycles that preceded it. Just over half of those changesets were pulled after the writing of [our first 5.5 merge-window summary](<https://lwn.net/Articles/806010/>). As is often the case later in the merge window, many of those changes were relatively boring fixes. There were still a number of interesting changes, though; read on for a summary of what happened in the second half of this merge window. 

#### Architecture-specific

  * The RISC-V architecture has gained support for the `[seccomp()](<http://man7.org/linux/man-pages/man2/seccomp.2.html>)` system call (including filtering with BPF). 
  * RISC-V systems without a memory-management unit are now supported. 
  * The xtensa architecture can now boot from execute-in-place kernels. 

#### Core kernel

  * The new `IORING_OP_CONNECT` command for [io_uring](<https://lwn.net/Articles/776703/>) allows `[connect()](<http://man7.org/linux/man-pages/man2/connect.2.html>)` calls to be performed asynchronously. 
  * After years of deprecation, the `[sysctl()](<http://man7.org/linux/man-pages/man2/sysctl.2.html>)` system call has been removed. 
  * Synthetic trace events can be created with the new [injection mechanism](<https://git.kernel.org/linus/6c3edaf9fd6a>). The use case appears to be testing of software that reacts to trace events. 

#### Filesystems and block I/O

  * The XFS "iomap" code has been moved into into the virtual filesystem layer, making this infrastructure available to other filesystems. The ext4 filesystem has been modified to use this code. The end result is simpler, more consistent, and hopefully less buggy direct I/O in a number of filesystems. 
  * The CIFS filesystem now supports the `[flock()](<http://man7.org/linux/man-pages/man2/flock.2.html>)` system call. CIFS has also gained [multichannel](<https://blogs.technet.microsoft.com/josebda/2012/06/28/the-basics-of-smb-multichannel-a-feature-of-windows-server-2012-and-smb-3-0/>) support, which should improve performance. 
  * The hugetlbfs filesystem now supports creating files with the `O_TMPFILE` option. 
  * The NFS client has gained support for cross-device offloaded copy operations — copying a file directly from one remote server to another. 

#### Hardware support

  * **Clock** : Qualcomm QCS404 Q6SSTOP clock controllers, Qualcomm SC7180 global clock controllers, Qualcomm MSM8998 graphics clock controllers, Ingenic X1000 clock generators, and Bitmain BM1880 clock controllers. 
  * **DMA** : NXP DPAA2 QDMA controllers, Milbeaut AHB and AXI DMA controllers, and Sifive PDMA controllers. 
  * **Miscellaneous** : Crane EL15203000 LED controllers, RDA Micro GPIO controllers, Broadcom XGS iProc GPIO controllers, Mellanox BlueField firmware boot control units, Amlogic G12 thermal sensors, Samsung EXYNOS5422 dynamic memory controllers, Qualcomm on-chip memory controllers, Marvell MMP3 USB PHYs, and NVIDIA Tegra30 external memory controllers. 

#### Security-related

  * The `seccomp()` [user-space notification mechanism](<https://lwn.net/Articles/756233/>) has gained a new return code, `SECCOMP_USER_NOTIF_FLAG_CONTINUE`, which instructs the kernel to allow the system call in question to continue executing. 

#### Internal kernel changes

  * It is now possible for kernel subsystems to set up their own tracing instances without worrying about interfering with any tracing done from user space. 
  * The [DMA-BUF heaps subsystem](<https://lwn.net/Articles/792733/>) — meant to serve as a replacement for the Android-specific ION allocator — was merged but then reverted. It seems that it lacks a demonstrated open-source user-space user as [required](<https://01.org/linuxgraphics/gfx-docs/drm/gpu/drm-uapi.html#open-source-userspace-requirements>) by the DRM subsystem. This feature will probably have to wait for 5.6. 
  * Most of the `ioctl()` compatibility code has been pushed out into the drivers that need it; this code should disappear entirely in the relatively near future. 
  * Code-testing coverage with kcov can now monitor execution by background kernel threads. Some special annotation is required; see [this commit](<https://git.kernel.org/linus/eec028c9386e>) for details. 

One item that (as expected) did not make it into 5.5 is the [WireGuard](<https://lwn.net/Articles/748582/>) virtual private network system. That long-awaited feature is coming soon, though: it has [already been merged into the networking tree](<https://lwn.net/ml/netdev/20191208.175209.1415607162791536317.davem@davemloft.net/>) for the 5.6 release. Meanwhile, the 5.5 kernel is now in the stabilization period where, with luck, all of the new bugs will be fixed. The final 5.5 release can be expected around the beginning of February 2020.
