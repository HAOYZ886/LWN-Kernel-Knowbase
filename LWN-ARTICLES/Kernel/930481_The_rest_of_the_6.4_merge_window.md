---
title: The rest of the 6.4 merge window
url: https://lwn.net/Articles/930481/
date: "May 8, 2023"
category: "Releases-6.4"
author: "By Jonathan Corbet May 8, 2023"
---

By **Jonathan Corbet**  
May 8, 2023

Linus Torvalds [released 6.4-rc1](<https://lwn.net/Articles/931180/>) and closed the merge window on May 7\. By that time, 13,044 non-merge changesets had found their way into the mainline repository for the 6.4 release. A little over 5,000 of those changesets came in after [our summary of the first half of the merge window](<https://lwn.net/Articles/930068/>) was written. Those changes brought a long list of new features and capabilities to the kernel. 

The most significant changes merged since the previous summary include: 

#### Architecture-specific

  * After many tries, the x86 [linear address masking](<https://lwn.net/Articles/902094/>) feature is finally supported by the kernel. In the end, only the `LAM_U57` mode (which allows the storing of six bits of metadata in an address value) is supported; the `LAM_U48` mode needs further consideration. 
  * The RISC-V architecture has gained support for the [Svnapot](<https://groups.google.com/a/groups.riscv.org/g/isa-dev/c/m-2tJYwJEf0/m/V5PdQ83JBQAJ>) extension. The purpose of this extension might not be obvious to all from its name; the "napot" part evidently stands for "naturally aligned power of two". It allows groups of 4K base pages to be tied together to create larger ("huge") pages. 
  * The RISC-V kernel can now be built as a PIE relocatable binary. 
  * RISC-V has also added a new system call, `riscv_hwprobe()`, to provide information about the hardware available on the running system. [This commit](<https://git.kernel.org/linus/ea3de9ce8aa2>) has some documentation. 
  * Also added to RISC-V is support for hibernation. 
  * S390 now supports the [STACKLEAK](<https://lwn.net/Articles/764325/>) security feature (initially merged for x86 in 2018), which clears the kernel stack prior to returning to user space. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Core kernel

  * The [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) system call has a new feature, `UFFD_FEATURE_WP_UNPOPULATED`, that allows unpopulated, anonymous memory areas to be write-protected. The new `UFFDIO_CONTINUE_MODE_WP` flag will cause page-table entries to be write-protected on minor faults. 
  * [Process-level samepage merging control](<https://lwn.net/Articles/928510/>) has been merged. This feature should make the kernel samepage merging feature more generally useful without compromising security. 
  * It is now possible for unprivileged processes to poll for [pressure-stall information](<https://lwn.net/Articles/759781/>). In the unprivileged case, the polling period must be a multiple of two seconds. 
  * User trace events, a mechanism that allows user-space processes to create trace events through the kernel, have [finally been merged](<https://lwn.net/Articles/927595/>) after a year of reworking. 

#### Filesystems and block I/O

  * The new `noswap` mount option for tmpfs filesystems will cause it to not use the system swap space for data storage. 
  * The XFS filesystem has seen more work on its online scrub mechanism; this is said to be the last piece needed before the online filesystem-check functionality can be merged. That will come in a future cycle; meanwhile, though, the [design documentation for XFS online fsck](<https://www.kernel.org/doc/html/next/filesystems/xfs-online-fsck-design.html>) has been merged. 
  * The ntfs3 `noacsrules` mount option [has been removed](<https://git.kernel.org/linus/267a36ba30a7>); it evidently never worked properly. Attempts to mount with that option will now fail with an error. 
  * The NFS server implementation has gained support for [RFC 9289](<https://datatracker.ietf.org/doc/rfc9289/>), which specifies remote procedure calls protected by TLS encryption. 

#### Hardware support

  * **Clock** : SkyWorks Si521xx PCIe clock generators, Sunplus SP7021 clocks, Loongson-2 clocks, MediaTek MT8188 clock controllers, Broadcom BCM63268 timer clock and reset controllers, StarFive JH7110 system clock controllers, Qualcomm SA8775P, SM6125, SM6375, and SM6115 graphics clock controllers, and Qualcomm SM7150, MSM8917/QM215, IPQ9574 and IPQ5332 global clock controllers. 
  * **GPIO and pin control** : Intel La Jolla Cove Adapter GPIO controllers, NXP S32G2 pin controllers, Qualcomm IPQ9574 and SM7150 pin controllers, and NVIDIA BlueField-3 pin controllers. 
  * **Industrial I/O** : Bosch Sensortec BMP580 pressure sensors, Texas Instruments ADS1100 and ADS1000 analog-to-digital converters, and ROHM BU27034 ambient light sensors. 
  * **Input** : NXP BBNSM power keys and Novatek NVT-ts touchscreens. 
  * **Miscellaneous** : Amlogic A1 SPI flash controllers, AMD composable DMA transfer (CDX) buses, non-volatile memory devices with "[layouts](<https://git.kernel.org/linus/266570f496b9>)", ASPEED AST2600 I3C controllers, Qualcomm PMIC flash LED modules, MediaTek MT6370 PMIC LED controllers, Rohm BD2606MVV LED drivers, Maxim 597x power switches, Renesas RZ/G2L MTU3a timers, Apple pulse-width modulators, and StarFive JH7100 and JH7110 watchdog timers. 
  * **Sound** : Freescale CPM time-slot assigners, Freescale CPM QUICC multichannel controllers, Cirrus Logic CS35L56 codecs, Analog Devices MAX98363 Soundwire speaker amplifiers, Realtek RT712 SDCA DMIC codecs, and AMD SoundWire managers. 
  * **USB** : UCSI Qualcomm PMIC GLINK interfaces. 
  * **Also** : the process of removing the kernel's ancient PCMCIA drivers [has begun](<https://git.kernel.org/linus/9b12f050c76f>). 

#### Miscellaneous

  * A lengthy series removing `MODULE_LICENSE()` declarations from code that cannot be built as a module has been merged. Getting there was a long story; see [this article](<https://lwn.net/Articles/927569/>) and [this merge message](<https://git.kernel.org/linus/b6a7828502dc>) for details. 
  * The new virtual file `/sys/kernel/tracing/touched_functions` will provide a list of all kernel functions that have been traced or had a BPF program attached to them. This information is expected to be most useful to developers debugging problems (possibly) related to tracing or a BPF program. 
  * New Rust language support includes the [pin-init core](<https://git.kernel.org/linus/90e53c5e70a6>) (meant to solve [the problem of initializing pinned data structures](<https://lwn.net/Articles/907876/>)), a number of locking primitives ([`LockClassKey`](<https://git.kernel.org/linus/6ea5aa08857a>), [`Lock` and `Guard`](<https://git.kernel.org/linus/76d4bd591e11>), [`Mutex`](<https://git.kernel.org/linus/6d20d629c6d8>), and [`SpinLock`](<https://git.kernel.org/linus/c6d917a498bf>)), the [`ARef`](<https://git.kernel.org/linus/f1fbd6a864d4>) type for reference-counted objects, [`Task`](<https://git.kernel.org/linus/313c4281bc9d>) as a representation of the kernel's `task_struct`, [`LockedBy`](<https://git.kernel.org/linus/7b1f55e3a984>) for data protected by an external lock, condition variables with [`CondVar`](<https://git.kernel.org/linus/19096bce8157>) (implemented using a wait queue), a [`UAPI`](<https://git.kernel.org/linus/4e1746656839>) crate for code dealing with user space, and a set of [`ioctl()`-number manipulation functions](<https://git.kernel.org/linus/ea76e08f4d90>). 
  * There is, as usual, [a long list](<https://git.kernel.org/linus/f085df1be60a>) of changes to the perf tool, including the ability to filter events with BPF, improved lock-contention reporting, a new reference-count checking infrastructure, and more. 

#### Virtualization and containers

  * There is a new set of hypercalls defined to enable Hyper-V guests to access pass-through PCI devices. The kernel has also gained initial support for Hyper-V [virtual trust levels](<https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/tlfs/vsm#virtual-trust-level-vtl>). 
  * KVM has a new framework for handling [SMCCC](<https://developer.arm.com/Architectures/Secure%20Monitor%20Calling%20Convention>) requests in user space. This, it is hoped, will enable the implementation of many virtualization-related operations in user space rather than adding more code to the kernel. 

#### Internal kernel changes

  * The way memory for loadable modules is managed within the kernel [has changed significantly](<https://git.kernel.org/linus/ac3b43283923>). The new `module_memory` mechanism increases the flexibility of memory management, with eventual improvements in both security and performance expected. 
  * The `DEFINE_SEMAPHORE()` macro now requires an argument for the initial value of the semaphore; see [this article](<https://lwn.net/Articles/928026/>) for more information on this change. 
  * The meaning of `MAX_ORDER`, which defines the size of the largest possible contiguous memory allocation in the kernel, has changed. While it was previously exclusive (the largest size was actually `MAX_ORDER-1`), it is now inclusive. The old meaning led to numerous subtle bugs that were fixed in the change. 
  * The [per-VMA locks series](<https://lwn.net/Articles/906852/>) has been [merged](<https://git.kernel.org/linus/0b6cc04f3db3>); it should bring improved scalablity to the memory-management subsystem. 

One significant feature that did not make it was [user-space shadow stacks for the x86 architecture](<https://lwn.net/Articles/926649/>). Torvalds [meditated](<https://lwn.net/ml/linux-kernel/CAHk-=whn3F1k263SZNUVQK195tcCMAo5E_WbmjUE0qFC5rWg=w@mail.gmail.com/>) on the pull request for almost the entire merge window before [finding a bug](<https://lwn.net/ml/linux-kernel/CAHk-=wiuVXTfgapmjYQvrEDzn3naF2oYnHuky+feEJSj_G_yFQ@mail.gmail.com/>) and concluding that he did not want to pull it without some significant changes. 

At this point, the merging of new features for 6.4 is complete, and the time has come to stabilize all of this work for the final release. Assuming that there are no surprises, that final release can be expected on June 25 or July 2.
