---
title: The rest of the 6.19 merge window
url: https://lwn.net/Articles/1049424/
date: "December 14, 2025"
category: "Releases-6.19"
author: "By Jonathan Corbet December 14, 2025"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 14, 2025

Linus Torvalds [released 6.19-rc1](<https://lwn.net/Articles/1050380/>) and closed the 6.19 merge window on December 14 (Japan time), after having pulled 12,314 non-merge commits into the mainline. Over 8,000 of those commits came in after [our first 6.19 merge-window summary](<https://lwn.net/Articles/1048869/>) was written. The second part of the merge window was focused on drivers, but brought in a number of other changes as well. 

The most significant changes pulled in the latter part of this merge window include: 

#### Architecture-specific

  * User-mode Linux has gained support, finally, for multiple processors. This support is limited in 6.19, though, in that threads within a single process cannot run concurrently. 
  * Support for the LoongArch32 subarchitecture has been merged, but it cannot actually be built until the toolchains catch up. 

#### Core kernel

  * There is now generic support for the management of page tables for I/O memory-management units; see [Documentation/driver-api/generic_pt.rst](<https://docs.kernel.org/next/driver-api/generic_pt.html>) for more information. 
  * System-call trace events are now able to read user-space buffers (file names, for example) and include them in the trace output. 
  * Guard pages are now specially marked in the `/proc/_PID_ /smaps` file; see [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5dba5cc2e0ff>) for details. 
  * The kernel is now able to manage transparent huge pages in device-private memory. See [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d245f9b4ab80>) for more information. 
  * The [zram](<https://docs.kernel.org/admin-guide/blockdev/zram.html>) device has gained support for writeback batching, improving performance. 
  * The [live update orchestrator](<https://lwn.net/Articles/1033364/>), which allows the kernel to be replaced on a running system, has been merged. See [this changelog](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9e2fd062fa17>) and [Documentation/core-api/liveupdate.rst](<https://docs.kernel.org/next/core-api/liveupdate.html>) for more information. 

#### Filesystems and block I/O

  * Caching of data from direct-I/O operations on NFS filesystems can now be disabled, further reducing the client-side cost of large I/O operations. [This documentation commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fa8d4e6784d1>) contains more information and details on how to use the new mode. 

#### Hardware support

  * **Clock** : Rockchip RV1126B and RK3506 clock controllers, Qualcomm IPQ5424 NSS clock controllers, Qualcomm SM8750 video clock controllers, Andes ATCRTC100 realtime clocks, NVIDIA VRS10 realtime clocks, and Apple Mac system management controller realtime clocks. 
  * **GPIO and pin control** : NXP QIXIS FPGA GPIO controllers, Intel Elkhart Lake PSE GPIO controllers, MediaTek MT6878 pin controllers, Qualcomm Kaanapali pin controllers, Microchip pic64gx gpio2 pin controllers, Microchip Polarfire pin controllers, and Cix Sky1 pin controllers. 
  * **Graphics** : Freescale i.MX8MP HDMI PAI bridges, Sharp LQ079L1SX01 panels, Synopsis Designware QP CEC interfaces, Arm Ethos-U65/U85 NPUs, Samsung S6E3FC2X01 DSI panels, Synaptics TDDI display panels, and LG LD070WX3 MIPI DSI panels. 
  * **Hardware monitoring** : MPS VR mp9945, mp2925 and mp2929 monitoring controllers, Analog Devices MAX17616/MAX17616A current limiters, Aosong dht20 temperature and humidity sensors, ST Microelectronics TSC1641 16-bit high-precision power monitors, and Apple system management controllers. 
  * **Industrial I/O** : Renesas RZ/T2H / RZ/N2H analog-to-digital converters, InvenSense ICM-456xx I2C, IC3, and SPI interfaces, Analog Devices MAX14001/MAX14002 analog-to-digital converters, Bosch SMI330 I2C and SPI interfaces, Aosong adp810 differential pressure and temperature sensors, and Renesas RZ/N1 analog-to-digital converters. 
  * **Media** : Sony IMX111 sensors, ARM Mali-C55 image signal processors, Renesas RZ/V2H(P) input video control blocks, and Rockchip camera interfaces. 
  * **Miscellaneous** : Airoha pulse-width modulators, T-HEAD TH1520 pulse-width modulators (Rust driver), MediaTek MT6316 and MT6363 SPMI PMIC regulators, FitiPower FP9931/JD9930 EPD regulators, NXP PF1550 PMICs, Microchip FPGA CoreSPI controllers, MediaTek MFlexGraphics power-domain controllers, Awinic AW99706 backlight controllers, ROHM BD71828 and BD71815 PMIC charger controllers, SpacemiT P1 poweroff and reset controllers, Richtek RT9756 smart cap divider chargers, Renesas RZ/G3S PCIe host controllers, NXP S32G PCIe host controllers, SpacemiT K1 PCIe host controllers, Qualcomm TC9563 PCIe switch power controllers, Broadcom next generation 50/100/200/400/800 gigabit RoCE HCAs, ESWIN SoC reset controllers, HiSilicon Hydra home agents, AMD Versal Gen 2 UFS controllers, Renesas Window WWDT watchdog timers, Qualcomm KAANAPALI interconnects, QNAP MCU EEPROMs, KEBA 8250 UARTs, Loongson 8250 based serial ports, and Ayaneo embedded controllers. 
  * **Sound** : Intel Novalake audio subsystems, Spacemit K1 I2S controllers, Cirrus Logic CS530x analog to digital converters, and CIX IPBLOQ HD audio interfaces. 
  * **USB** : Apple Silicon DWC3 platform controllers and Renesas RZ/G3E USB 3.0 PHYs. 

#### Miscellaneous

  * The `perf` tool has, among other things, gained support for unified event and metric descriptions in the JSON format and [deferred unwinding](<https://lwn.net/Articles/1029189/>) of user-space call stacks. See [this merge message](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9e906a9dead1>) for more information. 

#### Virtualization and containers

  * The [`guest_memfd()`](<https://lwn.net/Articles/1016133/>) implementation has gained support for NUMA policies, allowing hypervisors to set policies on where memory should be allocated. See [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ed1ffa810bd6>) for a little more information. 
  * The kernel has gained support for PCIe link encryption and device authentication; this allows confidential-computing guests to maintain encrypted communications with PCI devices and to ensure that they are talking to the devices they think they are. From [this merge message](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=249872f53d64>): 

> Linux gets a link encryption facility which has practical benefits along the same lines as memory encryption. It authenticates devices via certificates and may protect against interposer attacks trying to capture clear-text PCIe traffic. 

  * The HyperV "confidential VMBus" mechanism is another mechanism for confidential communication between guests and devices; see [this documentation commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=92c7053b44b3>) for more information. 

#### Internal kernel changes

  * The new `UT=1` build parameter will cause a warning to be emitted for each tracepoint that is declared but never used. Since each tracepoint consumes about 5KB of memory, there is value in removing the ones that are not actually useful. The intent is to make this warning the default once all of the existing unused tracepoints have been cleaned up. 
  * The internals of the `vmalloc()` allocator have been reworked to enable allocations to be safely made in atomic contexts (`GFP_ATOMIC` and such). As always, atomic allocations have a higher probability of failing. 
  * There is now support for module parameters for loadable modules written in Rust. 
  * The "Terminus 10x18" console font, meant to improve readability on mid-resolution (1440x900) laptop screens, has been added. 
  * This development cycle removed 98 exported symbols and added 483 new ones. See [this page](<https://lwn.net/Articles/1050382/>) for the full list. 
  * There are seven new kfuncs in 6.19: `__scx_bpf_dsq_insert_vtime()`, `__scx_bpf_select_cpu_and()`, `bpf_dynptr_from_file()`, `scx_bpf_dsq_insert___v2()`, `scx_bpf_dsq_peek()`, `scx_bpf_task_set_dsq_vtime()`, and `scx_bpf_task_set_slice()`. 

The time has now come to stabilize all of this work for the 6.19 release, which can be expected on February 1, 2026.
