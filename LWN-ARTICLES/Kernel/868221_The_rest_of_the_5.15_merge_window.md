---
title: The rest of the 5.15 merge window
url: https://lwn.net/Articles/868221/
date: "September 13, 2021"
category: "Releases-5.15"
author: "By Jonathan Corbet September 13, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 13, 2021

Linus Torvalds [released 5.15-rc1](<https://lwn.net/Articles/869030/>) and closed the merge window for this release on September 12; at that point, 10,471 non-merge changesets had found their way into the mainline repository. Those changesets contain a lot of significant changes and improvements. Read on for a summary of what came into the mainline in the roughly 7,000 changesets pulled since our [first-half summary](<https://lwn.net/Articles/867821/>) was written. 

#### Architecture-specific

  * As expected, [scheduling support](<https://lwn.net/Articles/838339/>) for asymmetric Arm systems has been merged. See [this documentation patch](<https://git.kernel.org/linus/702f43872665>) for details. 
  * Arm64 kernels can now be built as Hyper-V clients. 
  * The ARC architecture now supports three-level and four-level page tables; this is evidently a step toward the eventual support for 64-bit processors. 

#### Core kernel

  * The [`printk()` indexing](<https://lwn.net/Articles/857148/>) patch set has been merged. This feature allows the extraction of a list of all messages that can be emitted by `printk()`; user-space monitoring utilities can use this list to detect changes. 
  * Memory usage accounting for memory control groups has been extended to track many more in-kernel data structures, including those created for polling, signal handling, namespaces, and more. Some of those changes have since been reverted, though, due to the performance regression they caused. 
  * The memory-management subsystem has gained the ability to demote pages to a slower NUMA node in favor of reclaiming them entirely. This mechanism can move less-used pages to nodes hosting nonvolatile memory, which is better than discarding them. This is one piece of the tiered memory-management scheme described in [this article](<https://lwn.net/Articles/857133/>). 
  * The [multi-preference memory policy](<https://lwn.net/Articles/862707/#many>) patch has been merged; it allows a process to request that memory pages be allocated on a given set of NUMA nodes. 
  * The new [`process_mrelease()` system call](<https://lwn.net/Articles/864184/>) allows a supervisor process to hasten the reclaim of memory from an exiting process. 
  * As described in [this article](<https://lwn.net/Articles/866493/>), the code implementing the `MAP_DENYWRITE` option to `mmap()` has been removed. As a consequence, the number of situations where a write to a file will be denied with an `ETXTBSY` error has been reduced; see [this commit](<https://git.kernel.org/linus/49624efa65ac>) for a concise list. 
  * "Event probes" are a new type of kernel probe that attach to an existing trace event but which contain their own format string; they can be used to provide a new view of an existing tracepoint. [This commit](<https://git.kernel.org/linus/7491e2c44278>) contains a brief description of how they are used. 
  * The [DAMON subsystem](<https://lwn.net/Articles/812707/>) has been merged. DAMON allows the monitoring and optimization of memory management from user space. [This documentation patch](<https://git.kernel.org/linus/c4ba6014aec3>) contains more information. 

#### Filesystems and block I/O

  * It is no longer possible to turn off disk quota accounting in mounted XFS filesystems. _Enforcement_ can be turned off, but tracking will continue to happen under the hood. From [the pull request](<https://git.kernel.org/linus/90c90cda05ae>): ""We've tried to do this in a careful enough way that there shouldn't be any user visible effects aside from quotaoff no longer randomly hanging the system"". 
  * XFS now supports filesystems containing dates after 2038. This support has been present for a while but is no longer considered experimental. 
  * The EROFS filesystem now supports direct I/O on uncompressed files. 
  * Overlay filesystems now properly handle the immutable, append-only, sync, and noatime attributes. 
  * A new NTFS filesystem implementation, said to be far better than the kernel's existing version, has been merged. There were [some questions](<https://lwn.net/Articles/866112/#ntfs>) about whether this filesystem would make it into 5.15, but it seems that the doubts felt by some filesystem maintainers have been adequately addressed. 

#### Hardware support

  * **Clock** : Qualcomm SC7280 clock controllers, Qualcomm MSM8953 global clock controllers, Qualcomm MSM8994 multimedia clock controllers, Qualcomm SM6350, SM6115 and SM4250 global clock controllers, and MediaTek MT8192 clock controllers. 
  * **Graphics** : Innolux EJ030NA 320x480 LCD panels, Widechips WS2401 DPI panels, Samsung ATNA33XC20 eDP panels, and Ilitek ILI9341 240x320 QVGA panels. 
  * **Industrial I/O** : Sensirion SGP40 gas sensors, Analog Devices AD5110 digital potentiometers, and Renesas RZ/G2L analog-to-digital converters. 
  * **Media** : Amlogic Meson IR transmitters, Sony imx335 and imx412 image sensors, and OmniVision OV9282 sensors. 
  * **Miscellaneous** : Xilinx Versal FPGAs, General Electric ACHC microcontrollers, Nintendo Wii consoles, Qualcomm SC8180X interconnects, Realtek RTL8188EU Wireless LAN NICs (replacement driver, still in staging), Intel IXP4xx expansion bus interfaces, Renesas RZ/G2L USB/PHY controllers, Apple DART I/O memory-management units, Renesas I2C-controlled synchronization management units, MediaTek CPU-frequency controllers, Renesas RZ/G2L DMA controllers, and NVIDIA Tegra30 thermal sensors. 
  * **PCI** : Rockchip DesignWare PCIe controllers, Toshiba Visconti PCIe controllers, and Keem Bay PCIe controllers. 
  * **Pin control** : Qualcomm 9607 pin controllers, Qualcomm SM6115 and SM4250 pin controllers, STMicroelectronics STM32MP135 pin controllers, NXP i.MX8ULP pin controllers, Intel Keem Bay pin controllers, and Renesas RZ/G2L pin and GPIO controllers. 
  * **Sound** : Renesas RZ/G2L serial sound interfaces. 

#### Security-related

  * The kernel can now be built with the GCC `-fzero-call-used-regs=used-gpr` option; in English, that means that any registers used within a function will be zeroed just before returning from that function. This helps to prevent data from leaking out of functions; it also, as Kees Cook [noted](<https://git.kernel.org/linus/a82adfd5c7cb>) in the changelog, ""helps reduce the number of useful ROP [return-oriented programming] gadgets in the kernel image by about 20%"". 

#### Virtualization and containers

  * The new, fully undocumented gpio-virtio driver allows guests to access GPIO lines provided by the host system. 
  * There is a new driver framework called "VDUSE" that allows the implementation of virtual block devices in user space, with a Virtio transport to the guest. See [this documentation commit](<https://git.kernel.org/linus/7bc7f61897b6>) for some more information. 

#### Internal kernel changes

  * [Restricted DMA pools](<https://lwn.net/Articles/841916/>) can be used to place limits on the pages a DMA-capable device can access on systems without an I/O memory-management unit. 
  * Kernel builds will now use the `-Werror` flag by default, causing the build to fail in the presence of compiler warnings. There is a new `WERROR` configuration variable that can be used to turn this option off; it is meant to be used in cases where a very new (or old) compiler is generating spurious warnings. This change is causing build failures for some but, so far, Torvalds [is unsympathetic](<https://lwn.net/ml/linux-next/CAHk-=wgSzN4p0EW2LkA0YkRBjkOXpdaiLZMez=h=3d4Q2Gjp3A@mail.gmail.com/>): 

> So the fact that *I* require a clean build, and generally have to reject a couple of build requests every merge window because of that clearly doesn't mean that we actually have clean builds. 
> 
> I've had that "no warnings" requirement for years. And that means that *my* build has been clean for years. 
> 
> I want *all* builds to be clean, or at least as far as possible. 

Even so, Torvalds eventually applied [this patch](<https://git.kernel.org/linus/b339ec9c229a>) setting the `WERROR` configuration variable to match the value of `COMPILE_TEST` to minimize the pain felt by kernel builders worldwide. [Yet another patch](<https://git.kernel.org/linus/49832c819ab8>) silences the warning GCC issues when the name `main` is used for anything but a function. 
  * When the Clang compiler is used to build the kernel, the LLVM integrated assembler will also be used by default. 
  * The SLUB allocator has seen a significant rewrite to minimize its disabling of interrupts and make it compatible with realtime preemption. See [this changelog](<https://git.kernel.org/linus/cc09ee80c3b1>) for some more information. 
  * It is no longer possible to create executable mappings with `ioremap()`. 
  * Much of the kernel documentation has been translated into traditional Chinese. 

Notably not pulled was the folio patch set; see [this article](<https://lwn.net/Articles/868598/>) for more information on that subject. 

In general, this merge window has been relatively difficult — or, as Torvalds [put it](<https://lwn.net/ml/linux-kernel/CAHk-=wguv1zB0h99LKH1UpjNvcg7tsckE_udYr3AP=2aEUdtwA@mail.gmail.com/>): ""This has not been one of those smooth merge windows that we occasionally have"". Hopefully the stabilization period will be a bit less rough. If all goes well and the nine-week schedule sticks, we'll have the final 5.15 release as an extra Halloween treat on October 31.
