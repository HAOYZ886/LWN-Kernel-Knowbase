---
title: The rest of the 5.2 merge window
url: https://lwn.net/Articles/788532/
date: "May 20, 2019"
category: "Releases-5.2"
author: "By Jonathan Corbet May 20, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 20, 2019

By the time Linus Torvalds released the [5.2-rc1 kernel prepatch](<https://lwn.net/Articles/788857/>) and closed the merge window for this development cycle, 12,064 non-merge changesets had been pulled into the mainline repository — about 3,700 since [our summary of the first "half"](<https://lwn.net/Articles/787963/>) was written. Thus, as predicted, the rate of change did slow during the latter part of the merge window. That does not mean that no significant changes have been merged, though; read on for a summary of what else has been merged for 5.2. 

#### Architecture-specific

  * The PowerPC architecture can now take advantage of hardware support to prevent the kernel from accessing user-space data in unintended ways. 
  * 32-Bit PowerPC now has support for [KASAN](<https://lwn.net/Articles/612153/>). 
  * Mitigations for the Intel microarchitectural data sampling vulnerabilities have been merged. See [this page from the kernel documentation](<https://www.kernel.org/doc/html/latest/x86/mds.html#mds>) for a fairly detailed description of the problem, and [this page](<https://www.kernel.org/doc/html/latest/admin-guide/hw-vuln/mds.html>) for mitigation information. 

#### Core kernel

  * There is finally a freezer for the control-group version-2 implementation. It differs from the v1 freezer in that it puts each affected process into a stopped state rather than an uninterruptible sleep; that allows those processes to be operated on (killed, traced, moved to another group) while the group is frozen. See [this commit](<https://git.kernel.org/linus/afe471ea2d6f>) for the documentation update. 
  * The new `vm.unprivileged_userfaultfd` sysctl knob controls whether unprivileged users can use the [`userfaultfd()`](<http://man7.org/linux/man-pages/man2/userfaultfd.2.html>) system call. The default is to allow unprivileged access (which is consistent with current kernels). 
  * [Pressure stall monitors](<https://lwn.net/Articles/775971/>), which allow user space to detect and respond quickly to memory pressure, have been added. See [this commit](<https://git.kernel.org/linus/0e94682b73bf>) for documentation and a sample program. 
  * The tracing subsystem exports a new virtual file, `tracing/error_log`, where the more complex tracing operations can place error messages when things go wrong. 
  * The `/proc/slab_allocators` file turned out to have yet another set of bugs. Since it clearly hasn't worked correctly for years and nobody has complained, this file has been removed. 

#### Filesystems and block layer

  * The handling of soft mounts in NFS v4.0 has been improved, with more accurate timeout handling, faster failover, and a new `softerr` mount option that can change the error code for timed-out operations to `ETIMEDOUT`. 
  * The old `nfsdcld` (NFS client-tracking daemon) API has been resurrected as a way of allowing NFS servers to properly track client state over a reboot. If a daemon is running, it takes over the role of the [`nfsdcltrack` helper](<http://man7.org/linux/man-pages/man8/nfsdcltrack.8.html>); the intent is to create a solution that works better in a namespaced environment. 
  * There is a new device-mapper target called `dm-dust`; it can be used to simulate bad blocks in the underlying device. See [Documentation/device-mapper/dm-dust.txt](<https://lwn.net/Articles/788791/>) for details. 

#### Hardware support

  * **Clock** : ASPEED realtime clocks, MediaTek MT8183 and MT8516 clocks, Qualcomm QCS404 Turing clock controllers, Cirrus Logic Lochnagar clock controllers, and SiFive FU540 SoC power reset clock interfaces. 
  * **Input** : generic GPIO-controllable vibrators, Azoteq IQS550/572/525 trackpad/touchscreen controllers, and Microchip AT42QT1050 touch sensors. 
  * **Miscellaneous** : AMD MP2 PCIe I2C adapters, Marvell Armada 37xx rWTM BIU mailbox controllers, NXP i.MX TPM pulse-width modulators, Mellanox BlueField SoC GPIO controllers, ROHM BD70528 PMIC watchdog timers, NXP IMX SC watchdog timers, Maxim MAX77650/77651 power-management ICs, STMicroelectronics multi-function eXpanders, Ingenic JZ47xx SoCs battery monitors, Microchip UCS1002 USB port power controllers, and Xilinx ZynqMP FPGA managers. 

#### Internal kernel changes

  * The [`FOLL_LONGTERM` flag](<https://lwn.net/Articles/784574/#fl>) has been added to `get_user_pages()`; this is a part of the bigger effort to solve the problems with that interface and long-term mappings. 
  * Two new functions have been added to ease the task of mapping kernel memory into a user-space address range. `vm_map_pages()` and `vm_map_pages_zero()` will map a set of pages into a VMA; they differ in that the latter function ignores the `vm_pgoff` offset in the VMA. 
  * Code coverage analysis with `gcov` is now supported on Clang-compiled kernels. 
  * There have been significant changes to the implementation of `vmalloc()` that improve performance considerably; see [this commit](<https://git.kernel.org/linus/68ad4a330433>) for details. 

Barring surprises (and there have not been many surprises in recent years), the 5.2 kernel will be released on July 7 or 14.
