---
title: The first half of the 5.17 merge window
url: https://lwn.net/Articles/880909/
date: "January 13, 2022"
category: "Releases-5.17"
author: "By Jonathan Corbet January 13, 2022"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 13, 2022

As of this writing, just short of 7,000 non-merge commits have been pulled into the mainline kernel repository for the 5.17 release. The changes pulled thus far bring new features across the kernel; read on for a summary of what has been merged during the first half of the 5.17 merge window. 

#### Architecture-specific

  * The arm64 architecture has gained support for the [kernel concurrency sanitizer](<https://www.kernel.org/doc/html/latest/dev-tools/kcsan.html>) (KCSAN). 
  * 32-Bit Arm systems now support [KFENCE](<https://docs.kernel.org/dev-tools/kfence.html>). 
  * The boot-time memtest memory tester is now available on the m68k architecture. 
  * The new "AMD P-State" subsystem is a power-control mechanism for upcoming AMD processors that, it is said, offers significantly better performance. See [this documentation commit](<https://git.kernel.org/linus/c22760885fd6>) for more information. 

#### Core kernel

  * The [`bpf_loop()` helper](<https://lwn.net/Articles/877062/>) is an alternative way of implementing (some) loops in BPF programs; it can improve performance and ease the task of getting loops past the BPF verifier. 
  * The ["compile once/run everywhere" (CO-RE) mechanism](<https://nakryiko.com/posts/bpf-core-reference-guide/>), formerly implemented in user space, now runs within the kernel. This is a step toward the eventual implementation of signed BPF programs and also makes BPF functionality more readily available to languages like Go. 
  * The scheduler now tracks forced-idle time — the time that an SMT sibling processor is forced into the idle state as the result of [core-scheduling](<https://lwn.net/Articles/861251/>) constraints. This information, which can be used to evaluate the cost of enabling core scheduling, can be found in `/proc/_PID_ /sched`. 
  * The `RCU_FAST_NO_HZ` configuration option, meant for advanced tweaking of the RCU algorithm on tickless CPUs, [has been removed](<https://git.kernel.org/linus/e2c73a6860bd>). It seems that no actual users of this feature could be found. 

#### Filesystems and block I/O

  * [ID-mapped filesystem mounts](<https://lwn.net/Articles/837566/>) can now be made on top of filesystems that are, themselves, ID-mapped. [This merge commit](<https://git.kernel.org/linus/5dfbfe71e324>) has some more information. 

#### Hardware support

  * **Graphics** : the direct rendering subsystem has gained support for electronic privacy screens, as found on various laptop models. Also: JDI R63452 Full HD DSI panels, Ilitek ILI9163 display panels, Novatek NT35950 DSI panels, Boe BF060Y8M-AJ0 panels, Sony Tulip Truly NT35521 panels, and R-Car DU MIPI DSI encoders. 
  * **Hardware monitoring** : Texas Instruments INA238 power monitors, ASUS WMI B550/X570 and X370/X470/B450/X399 hardware monitoring interfaces, Delta AHE-50DC fan control modules, Renesas RZ/G2L thermal sensors, MPS MP5023 hardware monitoring interfaces, and NZXT fan controllers. 
  * **Media** : STMicroelectronics STM32 Chrom-Art accelerators, Maxim MAX96712 quad GMSL2 deserializers, OmniVision OV5693 sensors, and various codecs with VP9 support. 
  * **Miscellaneous** : Apple PMGR power-state controllers, R-Car Gen4 system controllers, Samsung Exynos universal serial interfaces, StarFive JH7100 clock generators, StarFive JH7100 reset controllers, Marvell CN10K performance-monitoring units, HiSilicon PCIe performance-monitoring units, Marvell CN10K random number generators, Letsketch WP9620N tablets, Maxim MAX77976 battery chargers, Lenovo Yoga Book tablets, Siemens Simatic LED controllers, Siemens Simatic IPC watchdogs, Asus TF103C 2-in-1 keyboard docks, Renesas R-Car Gen3 and RZ/N1 NAND controllers, TI TPS68470 PMIC regulators, and Maxim MAX20086-MAX20089 camera power protectors. 
  * **Networking** : Engleder TSN endpoint Ethernet MACs, Microchip Lan966x network switches, Vertexcom MSE102x SPI interfaces, and Mellanox Spectrum-4 Ethernet switches. 
  * **Pin control** : Qualcomm SDX65 and SM8450 pin controllers, StarFive JH7100 pin controllers, NXP IMXRT1050 pin controllers, and Intel Thunder Bay pin controllers. 

#### Networking

  * The [reference-count tracking infrastructure](<https://lwn.net/Articles/877603/>) has been added. This mechanism should help developers track down the source of reference-count bugs. For now it is specific to the networking subsystem but should be relatively easily extended to other parts of the kernel. 
  * The new "converged security and management engine" module allows communication with the Intel management engine (the separate processor lurking within Intel CPUs) via WiFi. 
  * Support for offloading traffic-control actions to network devices has been added; some information can be found in [this commit](<https://git.kernel.org/linus/434ed2138994>). 
  * The [management component transport protocol](<https://en.wikipedia.org/wiki/Management_Component_Transport_Protocol>) (MCTP) is now supported over serial devices. MCTP support over SMBus was also merged but subsequently [reverted](<https://git.kernel.org/linus/2f6a470d6545>) after the I2C maintainer complained about not having been involved in the necessary I2C core changes. 

#### Security-related

  * The kernel's random-number generator has switched from the SHA1 hash algorithm to BLAKE2s, which is both faster and more secure. 

#### Virtualization and containers

  * User-mode Linux can now be booted with a devicetree blob, facilitating testing of driver code. 
  * The Xen USB virtual host driver allows access to USB devices to be passed through to Xen guests. 

#### Internal kernel changes

  * The `struct page` fields used by the slab allocators have been moved into a separate structure. An early version of this work was covered in [this article](<https://lwn.net/Articles/871982/>); it has since been pushed further by Vlastimil Babka. 
  * KCSAN has [gained the ability to detect some missing memory barriers](<https://lwn.net/Articles/877200/>). Some more information can be found in [this documentation commit](<https://git.kernel.org/linus/82eb6911d909>). 
  * The new gpio-sim module creates simulated GPIO chips for testing; see [this commit](<https://git.kernel.org/linus/cb8c474e79be>) for more information. 
  * The kernel is now built with `-Wcast-function-type`, which generates a warning when function pointers are cast to an incompatible type. This check is necessary to avoid setting off [control-flow integrity](<https://lwn.net/Articles/856514/>) alarms. 

There are currently about 4,000 changesets sitting in linux-next, suggesting that activity will be a bit slower for the rest of the merge window. That said, there are undoubtedly some interesting changes yet to land in the mainline; that should happen by the time the merge window closes on January 23\. Stay tuned for our second-half summary, which will arrive shortly after that date.
