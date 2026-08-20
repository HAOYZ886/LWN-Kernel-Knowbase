---
title: The rest of the 7.1 merge window
url: https://lwn.net/Articles/1067785/
date: "April 27, 2026"
category: "Releases-7.1"
author: "By Jonathan Corbet April 27, 2026"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 27, 2026

By the time Linus Torvalds released [7.1-rc1](<https://lwn.net/ml/all/CAHk-=wh7jmSh6bO5VL31hOC3HdTY0QAV-P1H4XZauwL2x=w35Q@mail.gmail.com/>) and closed the 7.1 merge window, 12,996 non-merge changesets had been pulled into the mainline repository; just over 9,000 of those arrived after [the first-half summary](<https://lwn.net/Articles/1067250/>) was written. These changes were more driver-oriented than those seen earlier, but still also included many new features across the kernel as a whole. 

Aside from the specific changes, it is worth noting that 2,011 developers contributed changes during the 7.1 merge window; 342 of those developers were first-time contributors to the kernel. The trend of increasing numbers of new developers coming into the community that was described in the [development statistics article for 7.0](<https://lwn.net/Articles/1066723/>) appears to be continuing. 

The most interesting changes added in the latter part of the 7.1 merge window include: 

#### Architecture-specific

  * The Alpha architecture now has [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>) support. 
  * High-memory support has been added to Loongarch, enabling 32-bit machines to use up to 2.25GB of physical memory. This change, of course, runs counter to the [long-term plan to remove high-memory support](<https://lwn.net/Articles/1051010/>) altogether. 
  * Execute-in-place support for RISC-V has been removed due to a seeming lack of users (and maintainers); [this merge message](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=feff82eb5f40>) has some more information. 

#### Core kernel

  * The [extensible scheduler class (sched_ext)](<https://lwn.net/Articles/922405/>) subsystem now has the beginning of support for [sub-schedulers](<https://lwn.net/Articles/1056014/>), meaning that each control group can run with its own custom CPU scheduler. As of 7.1, this implementation is not complete (it lacks the enqueue path in particular), but much of the needed core structure is there. See [this commit log](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ebeca1f930ea>) for some more information. 
  * The work to modernize and improve the kernel's swap subsystem continues with [the removal of the old swap map](<https://lwn.net/Articles/1057102/>). With luck, the only user-visible changes will be better performance and reduced memory usage by the swap subsystem itself. 
  * The [DAMON subsystem](<https://docs.kernel.org/next/mm/damon/index.html>) has gained the ability to support multiple tuning algorithms for active memory management. See [this changelog](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8719c59c4b92>) and [the DAMON documentation](<https://docs.kernel.org/next/mm/damon/design.html#aim-oriented-feedback-driven-auto-tuning>) for details. 
  * The tracing subsystem has gained the concept of "remote ring buffers" that can be used to obtain tracing data from, for example, virtual machines. There is some information on how they work in [Documentation/trace/remotes.rst](<https://docs.kernel.org/next/trace/remotes.html>). 
  * The kernel has long had a problem where a dying control group would hang around indefinitely, keeping a lot of memory pinned in the process; it was, for example, [discussed](<https://lwn.net/Articles/787614/>) at the 2019 Linux Storage, Filesystem, and Memory Management Summit, and [again](<https://lwn.net/Articles/895431/>) at the 2022 meeting. In 7.1, that problem has finally been solved thanks to [this patch series](<https://lwn.net/ml/all/cover.1768389889.git.zhengqi.arch@bytedance.com/>) from Qi Zheng and Muchun Song. Now, folios in memory will no longer pin their associated control group, allowing that group to be released more quickly once its final member has exited. 
  * User-space page-fault handling with [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) now works with [guest_memfd](<https://lwn.net/Articles/1016133/>) memory. 

#### Filesystems and block I/O

  * There is [a completely rewritten version of the NTFS filesystem](<https://lwn.net/Articles/1055062/#ntfs>) with full support for writes, a conversion to use [iomap](<https://docs.kernel.org/filesystems/iomap/>) internally, and more, along with a promise of more active maintenance. This implementation is an alternative to ntfs3, which has not progressed as quickly as some would have liked. 
  * The ntfs3 implementation has received [a small set of improvements](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a5d1079c28a5>) as well. 
  * The NFS server is now able, when the `sign_fh` mount option is used, to provide cryptographically signed file handles to clients as a way of protecting against handle-guessing attacks. There is some new documentation in [Documentation/filesystems/nfs/exporting.rst](<https://docs.kernel.org/next/filesystems/nfs/exporting.html#signed-filehandles>) describing this option. 
  * The new fs-dax char driver ([commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d5406bd458b0>)) provides a filesystem-oriented interface to direct-access (DAX) devices. It is a precondition for the eventual merging of the [famfs in-memory filesystem](<https://lwn.net/Articles/1068686/>). 
  * The Ceph filesystem can now collect per-subvolume I/O metrics; from [the commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b1137e0b3d4b>): ""This enables administrators to monitor I/O patterns at the subvolume granularity, which is useful for multi-tenant CephFS deployments."" 

#### Hardware support

  * **Clock** : Qualcomm GLYMUR graphics and video clock controllers, Rockchip RV1103B clock controllers, Tenstorrent Atlantis PRCM clock controllers, Qualcomm Eliza global, display, and TCSR clock controllers, ESWIN EIC7700 clock controllers, Qualcomm IPQ5210 global clock controllers, Qualcomm SM8750 graphics clock controllers, and Qualcomm Nord global and TCSR clock controllers. 
  * **GPIO and pin control** : Qualcomm Eliza pin controllers, Qualcomm Milos LPASS LPI pin controllers, Qualcomm IPQ5210 pin controllers, Qualcomm SDM670 LPASS LPI pin controllers, Qualcomm Hawi pin controllers, Realtek DHC 1625 pin controllers, and GPIO-driven Charlieplex keypads. 
  * **Graphics** : Verisilicon DC-series display controllers, T-Head TH1520 DesignWare HDMI bridges, Coreboot framebuffers, LXD M9189A MIPI-DSI LCD panels, Novatek NT37700F DSI panels, Atrix 4G and Droid X2 540x960 DSI video-mode panels, Lontium LT8713SX DP MST bridges, Himax HX83121A-based DSI panels, Ilitek ILI9806E-based RGB SPI panels, and Samsung S6E8FC0 DSI controllers. 

The graphics layer has also gained support for the [DRM-RAS subsystem](<https://docs.kernel.org/next/gpu/drm-ras.html>), which ""provides a standardized way for GPU/accelerator drivers to expose error counters and other reliability nodes to user space via Generic Netlink"". 
  * **Hardware monitoring** : Infineon XDPE1A2G7B and XDP720 hardware-monitoring controllers, LattePanda Sigma EC hardware monitors, Lenovo Yoga fan-speed monitors, Sony APS-379 power supplies, and Microchip Technology MCP9982 automotive temperature monitors. 
  * **Industrial I/O** : Qualcomm SPMI PMIC5 GEN3 analog-to-digital converters, Qualcomm Eliza interconnects, and STMicroelectronics VL53L1X time-of-flight ranger sensors. 
  * **Input** : Lenovo Legion Go S controllers. 
  * **Media** : OmniVision OV2732 sensors and Toshiba T4KA3 sensors. 
  * **Miscellaneous** : Black Sesame Technologies BST C1200 digital host controller interfaces, Andes QiLai PCIe controllers, ESWIN PCIe controllers, IBM internal shared memory (ISM) adapters, Cix Sky1 reset controllers, Microchip PolarFire SoC's GPIO IRQ multiplexers, Switchtec PSX/PFX switch DMA engines, ESWIN eic7700 SATA SerDes/PHYs, Loongson chain multi-channel DMA controllers, Apple SMC battery and power monitors, Samsung S2MU005 PMIC fuel gauges, Maxim MAX77759 battery chargers, and Tenstorrent Atlantis reset controllers. 
  * **Sound** : Cirrus Logic CS42L43B codecs. 
  * **USB** : Canaan USB2 PHYs. 

#### Miscellaneous

  * The [runtime verification subsystem](<https://lwn.net/Articles/857862/>) has gained some new tools, including [hybrid automata](<https://docs.kernel.org/next/trace/rv/hybrid_automata.html>), a [stall monitor](<https://docs.kernel.org/next/trace/rv/monitor_stall.html>) tool, and a [deadline monitor](<https://docs.kernel.org/next/trace/rv/monitor_deadline.html>). 
  * Updates to the `perf` tool include a new `comm_nodigit` sort key to combine threads whose name differs only by a trailing number, a `--pmu-filter` option to select a specific performance monitoring unit, and more; see [this merge log](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=df8f6181ab57>) for the full list. 
  * The increase in LLM-driven security-bug reports is motivating the removal of a number of old and (seemingly) unused drivers and network protocols. Removals so far affect [PCMCIA host-controllers](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b3c26ea81ccc>) and, via [this big pull request](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=64edfa65062d>), numerous networking drivers along with the ATM, ax25, amateur radio, ISDN, Bluetooth CMTP, and CAIF subsystems — over 140,000 lines of code in all. 

#### Security-related

  * The first-half summary noted that the libcrypto subsystem lacked documentation; that omission [has now been rectified](<https://origin.kernel.org/doc/html/next/crypto/libcrypto.html>). 

#### Virtualization and containers

  * [Protected KVM](<https://www.kernel.org/doc/html/next/virt/kvm/arm/pkvm.html>) for anonymous memory — where pages used by guests are unmapped from the host's virtual address space — is now supported to an extent on Arm CPUs. Full isolation from the host remains a distant goal, and this feature is not yet considered ready for general use. See [Documentation/virt/kvm/arm/pkvm.rst](<https://docs.kernel.org/next/virt/kvm/arm/pkvm.html>) for more information. 
  * The protected KVM and [nVHE](<https://developer.arm.com/documentation/102142/0100/Virtualization-host-extensions>) hypervisors can now pass trace data to the host via the new remote ring buffer tracing feature. 
  * KVM has gained support for the [ARM virtual generic interrupt controller v5](<https://docs.kernel.org/next/virt/kvm/devices/arm-vgic-v5.html>). 

#### Internal kernel changes

  * The [work to deprecate the `mmap()` `file_operations`](<https://lwn.net/Articles/1038715/>) continues with the addition of some new functionality and the conversion of a number of in-kernel users. There is [a new document](<https://docs.kernel.org/next/filesystems/mmap_prepare.html>) on how to convert existing code from `mmap()` to `mmap_prepare()`. 

All of this work now goes into the stabilization phase, where bugs are found and (hopefully) fixed. The most likely date for the release of the 7.1 kernel at the end of that process is June 14.
