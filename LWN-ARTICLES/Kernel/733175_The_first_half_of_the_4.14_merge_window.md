---
title: The first half of the 4.14 merge window
url: https://lwn.net/Articles/733175/
date: "September 8, 2017"
category: "Releases-4.14"
author: "By Jonathan Corbet September 8, 2017"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 8, 2017

As of this writing, just over 8,000 non-merge changesets have been pulled into the mainline kernel repository for the 4.14 development cycle. In other words, it looks like the pace is not slowing down for this cycle either. The merge window is not yet done, but quite a few significant changes have been merged so far. Read on for a summary of the most interesting changes entering the mainline in the first half of this merge window. 

Significant user-visible changes include: 

  * The [ORC unwinder](<https://lwn.net/Articles/728339/>) has been merged, supporting more reliable kernel tracebacks and live patching. The kernel also runs a bit faster when ORC is used instead of frame pointers. 

  * The control group [thread mode](<https://lwn.net/Articles/729215/>) patches have been merged. This paves the way for the CPU controller to finally appear under the version-2 interface, but that work has not been merged yet. It may, apparently, still happen during the 4.14 development cycle. 

  * The AMD [secure memory encryption](<https://lwn.net/Articles/686808/#sme>) feature is now supported. 

  * The RDMA subsystem has a new user-space API based on `ioctl()`. This API was [posted on the linux-api list](<https://lwn.net/Articles/733179/>) at the beginning of August but received no review comments, perhaps because it lacks any sort of documentation. Doug Ledford merged it, [saying](<https://lwn.net/Articles/733180/>) that ""it's encased in a Kconfig item that marks it experimental, so including it doesn't freeze it in stone"". Experience says that it could become frozen regardless of markings, though, if applications begin to depend on it; developers with an interest in this API might want to have a close look relatively soon. 

  * The `membarrier()` system call has gained [a new expedited option](<https://lwn.net/Articles/728795/>) that executes more quickly at the cost of creating inter-processor interrupts. 

  * The perf events subsystem continues to develop quickly; changes this time around include branch-type profiling and tracing support, the ability to visualize [fused instructions](<https://en.wikichip.org/wiki/macro-operation_fusion>), initial support for namespaces, and more; [this changelog](<https://git.kernel.org/linus/9657752cb5039c7498d4b27c4a75530f93b87d9b>) has an overview. 

  * The [lguest virtualization system](<https://lwn.net/Articles/218766/>) (which some of us still call the "rustyvisor") has been removed due to lack of interest and maintenance. 

  * The x86 architecture now supports [five-level page tables](<https://lwn.net/Articles/717293/>), allowing processors to manage up to 128PB of virtual address space on 4PB of physical memory. Surely nobody will ever need more memory than that. 

  * CPU-frequency governors [can now work across CPUs](<https://lwn.net/Articles/732740/>). This should lead to better power management, but also better responsiveness when the system's load changes. 

  * The [`MSG_ZEROCOPY` patches](<https://lwn.net/Articles/726917/>), adding zero-copy network transmission, have been merged. 

  * Progress toward [more scalable swapping](<https://lwn.net/Articles/717707/>) continues with work that delays the splitting of huge pages until after they have been swapped out. This is not the final step (which will be storing them as huge pages in the swap area), but it still brings a claimed 42% improvement in swap-out throughput. 

  * The new `MADV_WIPEONFORK` option for the `madvise()` system call causes the affected memory region to appear to be full of zeros in the child process after a fork. It differs from the existing `MADV_DONTFORK` in that the address range will remain valid in the child. 

  * New hardware support includes: 

    * **Audio** : Realtek RT274 codecs, Wolfson Microelectronics WM8524 codecs, and Cirrus Logic CS43130 codecs. 

    * **Cryptographic** : Allwinner Security System pseudo-random number generators, Microchip / Atmel elliptic curve crypto accelerators, AMD secure processors, STMicroelectronics STM32 hash accelerators, Freescale i.MX RNGC random number generators, and Axis ARTPEC-6/7 hardware crypto accelerators. 

    * **Graphics** : Pervasive Displays RePaper panels, Synopsys Designware MIPI DSI host DRM bridges, Synopsis Designware CEC interfaces, STMicroelectronics STM32 DSI controllers, and Sitronix ST7586 display panels. 

    * **Hardware monitoring** : IBM Common Form Factor power supplies, TI TPS53679 monitoring chips, and Lantiq CPU temperature sensors. 

    * **Industrial I/O** : Linear Technology LTC2471 and LTC2473 analog-to-digital controllers (ADCs), Diolan DLN-2 ADCs, Cirrus Logic EP93XX ADCs, AMS CCS811 VOC sensors, and Devantech SRF02 and SRF10 ultrasonic ranger sensors. 

    * **Media** : Omnivision OV5670 sensors, Analog Devices ADV748x decoders, ST STV0910 DVB-S/S2 demodulators, ST STV6111-based tuners, Amlogic Meson AO CEC interfaces, MaxLinear MxL5xx-based tuner-demodulators, GPIO and PWM-based infrared transmitters, ZTE ZX IR remote controls, and AMS AS3645A LED flash controllers. 

    * **Miscellaneous** : UniPhier AIDET interrupt controllers, Pi433 radio modules, Altera Arria-V/Cyclone-V/Stratix-V CvP FPGA managers, MediaTek MT6380 power-management ICs, MediaTek AHCI SATA controllers, and Renesas R-Car Gen3 SDHI DMA controllers. 

    * **Networking** : Hisilicon HNS3 Ethernet interfaces, Realtek RTL8822BE wireless network adapters, Adaptrum Anarion GMAC Ethernet controllers, Mellanox Technologies MLX5 SRIOV E-Switch switches, Rockchip Ethernet PHYs, Huawei PCIE network interfaces, and Marvell CP110 PHYs. 

    * **Pin control** : UniPhier PXs3 SoC pin controllers, NXP IMX7ULP pin controllers, Intel Denverton and Lewisburg pin controllers, Renesas R8A77995 pin controllers, Spreadtrum SC9860 pin controllers, TI TPS68470 GPIO controllers, and Cavium ThunderX/OCTEON-TX GPIO controllers. 

    * **USB** : Ralink USB PHYs and Atheros ath10k USB controllers. 

  * The IRDA (infrared devices) driver subsystem has been moved to the staging tree with the idea of deleting it entirely in the near future. All IRDA users should have moved to better alternatives some time ago. 

Changes visible to kernel developers include: 

  * For anybody who has been having a hard time building the kernel documentation: the new `sphinx-pre-install` script will examine the system and list the packages that should be installed to have a complete documentation toolchain. 

  * `spin_unlock_wait()` has been removed; its semantics were never entirely well defined and there did not appear to be any real need for it. 

  * Three architectures (ARM, ARM64, and x86) now check the value of the user-space address limit on return from system calls; this is meant to prevent [security holes](<https://lwn.net/Articles/722267/>) resulting from a failure to reset that value after it is changed. A new task flag (`TIF_FSCHECK`) is used to avoid slowing down the return on the bulk of system calls that don't call `set_fs()`. 

  * The [lockdep cross-release feature](<https://lwn.net/Articles/709849/>) has been merged; this will extend automatic lock checking to several use patterns where it was not possible before. 

  * The [fast reference-count overflow protection](<https://lwn.net/Articles/728675/>) mechanism has been disabled on x86 due to some unexplained warnings; presumably it will come back during the 4.14 cycle. 

  * The arm64 architecture now has support for [virtually mapped kernel stacks](<https://lwn.net/Articles/692208/>). 

  * The kernel has traditionally reserved 20 major numbers for dynamic device-number assignment, but that has proved to be too few on some systems. Starting with 4.14, a new range of 128 numbers starting at 511 and progressing downward has been set aside; they will be used after the original 20 have been exhausted. 

By the usual schedule, the 4.14 merge window should stay open through September 17, though that schedule has been known to vary at times. The actual 4.14 release is most likely to happen on November 5 or 12.
