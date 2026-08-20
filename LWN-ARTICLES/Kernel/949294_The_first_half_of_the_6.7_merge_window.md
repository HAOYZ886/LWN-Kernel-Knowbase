---
title: The first half of the 6.7 merge window
url: https://lwn.net/Articles/949294/
date: "November 3, 2023"
category: "Releases-6.7"
author: "By Jonathan Corbet November 3, 2023"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 3, 2023

As of this writing, 9,842 non-merge changesets have found their way into the mainline repository since the 6.7 merge window opened. Nearly a third of those consist of the entire bcachefs development history but, even discounting that, there has been a lot of material landing for the next release. Read on for a summary of the most interesting changes pulled so far in this development cycle. 

#### Architecture-specific

  * It is now possible to enable or disable 32-bit emulation on x86-64 kernels with the `ia32_emulation=` command-line parameter. This allows 32-bit emulation to be turned off where it is not needed. That, in theory, reduces the kernel's attack surface, since the 32-bit compatibility interfaces are seen as being less well tested than the rest of the kernel API. This option, though, allows the capability to be present for those who use it. The `IA32_EMULATION_DEFAULT_DISABLED` configuration option controls whether 32-bit emulation is enabled by default. 
  * S390x and 32-bit Arm systems now support the current set (cpuv4) of BPF instructions. 
  * After years of [discussion](<https://lwn.net/Articles/920259/>), support for the ia64 ("Itanium") architecture has been removed. Not everybody is happy about this decision, though, and Linus Torvalds has [indicated](<https://lwn.net/ml/linux-kernel/CAHk-=whFLZ67ffzt1juryCYcYz6eL_XjQF8WucDzwUR5H65+rA@mail.gmail.com/>) that he would be open to restoring ia64 support — but only after seeing it properly maintained out-of-tree for a year. 

#### Core kernel

  * The [futex2 API](<https://lwn.net/Articles/940944/>) has been merged, providing an alternative to the single, multiplexed [`futex()`](<https://man7.org/linux/man-pages/man2/futex.2.html>) system call. The new API also adds features for better performance on NUMA systems and support for sizes other than 32 bits. [**Update** : those features were not actually a part of this merge and will presumably show up in a future release; apologies for the error.] 
  * It is now possible to use [binfmt_misc](<https://docs.kernel.org/admin-guide/binfmt-misc.html>) to add new binary formats within unprivileged namespaces; see [this commit](<https://git.kernel.org/linus/21ca59b365c0>) for more information. 
  * A set of Rust bindings for workqueues has been added; [this commit](<https://git.kernel.org/linus/15b286d1fd05>) contains some examples of their use. 
  * [Cpusets](<https://docs.kernel.org/admin-guide/cgroup-v2.html#cpuset>) have a new "remote partition" mode that makes some configuration tasks easier; see [this documentation commit](<https://git.kernel.org/linus/efdf7532bd3d>) for more information. 
  * BPF programs can now make use of per-CPU kptrs; a small amount of information is available in [this changelog](<https://git.kernel.org/linus/1e4a6d975e5c>). 
  * Support for BPF exceptions (which are best thought of as a way to force an immediate exit from a BPF program) has been added. See [this article](<https://lwn.net/Articles/938435/>) and [this changelog](<https://git.kernel.org/linus/ec6f1b4db95b>) for more information. 
  * The io_uring subsystem now supports a number of new operations. [`IORING_OP_READ_MULTISHOT`](<https://git.kernel.org/linus/fc68fcda0491>) will perform multiple reads from a file descriptor until a buffer fills. [`IORING_OP_WAITID`](<https://git.kernel.org/linus/f31ecf671ddc>) is an asynchronous version of [`waitid()`](<https://man7.org/linux/man-pages/man2/wait.2.html>). [`SOCKET_URING_OP_GETSOCKOPT`](<https://git.kernel.org/linus/a5d2f99aff6b>) and [`SOCKET_URING_OP_SETSOCKOPT`](<https://git.kernel.org/linus/4232c6e349f3>) implement [`getsockopt()` and `setsockopt()`](<https://man7.org/linux/man-pages/man2/getsockopt.2.html>). 
  * Io_uring has also gained [support for futex operations](<https://git.kernel.org/linus/4de520f1fcef>), though only a subset of the futex API is implemented now. 

#### Filesystems and block I/O

  * The [fscrypt](<https://docs.kernel.org/filesystems/fscrypt.html>) subsystem can now encrypt data in units smaller than the filesystem block size; [this commit](<https://git.kernel.org/linus/5b1188847180>) includes some documentation on this feature. 
  * The Btrfs filesystem has added a new "stripe tree" data structure; its initial use is to implement RAID0 and RAID1 on zoned block devices, but it is expected to eventually address a number of longstanding problems with higher RAID levels in Btrfs in general. [This out-of-tree document](<https://lwn.net/ml/linux-btrfs/20230918140515.2597127-1-johannes.thumshirn@wdc.com/>) provides more information. 
  * Btrfs has also added "simple quotas", which address some of the performance problems experienced with full quota support. Simple quotas only track extents in the subvolume where they were created, resulting in a much simpler calculation that is, as a consequence, unable to account for shared extents. The feature is undocumented in-tree, but [this cover letter](<https://lwn.net/Articles/944371/>) gives an overview. 
  * The bcachefs filesystem has finally been merged, though marked as "experimental" for now. The merge contains nearly 2,800 commits, not a single one of which adds documentation. There is information on this filesystem at [bcachefs.org](<https://bcachefs.org/>). 
  * The kernel has gained support for [TLS encryption for NVMe-TCP](<https://lwn.net/Articles/942817/>). 

#### Hardware support

  * **Clock** : Cirrus Logic ep93xx timers, Amlogic S4 SoC PLL and peripheral clock controllers, TI TWL6032 clock controllers, Qualcomm SM8550 camera clock controllers, and Qualcomm SM4450 global clock controllers. 
  * **Graphics** : JDI LPM102A188A DSI panels, Raydium RM692E5-based DSI panels, and Solomon SSD132x OLED displays. 
  * **Miscellaneous** : Xilinx Versal DDR memory controllers, Analog Devices MAX77503 regulators, Mitsumi MM8013 fuel gauges, Qualcomm PM8916 BMS-VM fuel gauges, Qualcomm PM8916 linear battery chargers, Ampere Coresight performance monitoring units, Nuvoton NPCM BMC sdhci-pltfm controllers, and Qualcomm QSEECOM interfaces. 
  * **Networking** : Loongson1 GMAC Ethernet controllers, Intel data path function devices, digital phase-locked-loop controllers, I3C-connected MCTP devices, and Mediatek MT7925-based wireless interfaces. 

#### Miscellaneous

  * Rust 1.73.0 is now the version needed to build the Rust-for-Linux code. 

#### Networking

  * The [fair queuing packet scheduler](<https://man7.org/linux/man-pages/man8/tc-fq.8.html>) has gained a number of [performance improvements](<https://git.kernel.org/linus/b49a948568dc>): ""This series brings a 5% throughput increase in intensive tcp_rr workload, and 13% increase for (unpaced) UDP packets."" 
  * The TCP protocol can now optionally support microsecond-resolution timestamps on a per-route basis; [this changelog](<https://git.kernel.org/linus/bdf24b4bdfa5>) includes instructions on how to enable this feature. 
  * There is a new form of virtual network device where the transmit logic is entirely provided by a BPF program; [this changelog](<https://git.kernel.org/linus/35dfaad7188c>) has a bit more information. 
  * The TCP authentication option ([RFC 5925](<https://datatracker.ietf.org/doc/html/rfc5925>)) is now supported; it supersedes the older, MD5-based authentication mechanism. [This commit](<https://git.kernel.org/linus/7fe0e38bb669>) contains documentation on how TCP-AO works and how to use it. 

#### Virtualization and containers

  * The [iommufd subsystem](<https://docs.kernel.org/userspace-api/iommufd.html>) can now perform dirty-tracking for DMA operations. According to [the merge message](<https://git.kernel.org/linus/463f46e114f7>): ""This can be used to generate a record of what memory is being dirtied by DMA activities during a VM migration process. A VMM like qemu will combine the IOMMU dirty bits with the CPU's dirty log to determine what memory to transfer."" 

#### Internal kernel changes

  * There is a new "lightweight queue" implementation which is ""a FIFO single-linked queue that only requires a spinlock for dequeueing, which happens in process context. Enqueueing is atomic with no spinlock and can happen in any context."" There is no documentation outside of the kerneldoc comments in [the source](<https://git.kernel.org/linus/de9e82c355f2>)
  * Also added is "[objpool](<https://git.kernel.org/linus/b4edb8d2d464>)", which is ""a scalable implementation of high performance queue for object allocation and reclamation"". The usage of this feature can be seen in [this test module](<https://git.kernel.org/linus/92f90d3b0d5e>). 

There is still a fair amount of work sitting in linux-next, most of which can be expected to land in the mainline before the end of the merge window. That, in turn, should happen on November 12\. Keep an eye on LWN for our second-half summary once the merge window closes.
