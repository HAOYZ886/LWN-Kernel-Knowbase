---
title: "What's coming in 3.10, part 1"
url: https://lwn.net/Articles/548834/
date: "May 1, 2013"
category: "Releases-3.10"
author: "By Jonathan Corbet May 1, 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 1, 2013

As of this writing, nearly 5,100 non-merge changesets have been pulled into the mainline repository for the 3.10 development cycle. That is a pretty good pace, given that this merge window only opened on April 29\. A number of interesting new features have been added, with more to come. 

User-visible changes merged at the beginning of the 3.10 cycle include: 

  * The "ftrace" tracing facility has seen a number of improvements. At the top of the list, arguably, is the ability to establish multiple buffers for tracing information and to direct specific events to different buffers. Additionally, it is now possible set up a trigger that will enable or disable specific events when a given kernel function is called, and it is possible to get a stack trace when a given function is called. See [the improved ftrace.txt document](<https://lwn.net/Articles/548894/>) for lots of details. 

  * Control groups have seen a fair amount of work in this release, some of which is implementing the [improvements planned](<https://lwn.net/Articles/484251/>) over one year ago. The device and perf_event groups now offer full hierarchical support. There is a new mount option (with the unwieldy name "`__DEVEL__sane_behavior`") that tries to ensure more consistent hierarchy behavior across all groups; it is obviously meant for development rather than production, but it gives some clues about what is coming. 

  * Applications within control groups can now request [memory pressure notifications](<https://lwn.net/Articles/531077/>) when the system is running low on available memory. See the new section 11 at the end of [Documentation/cgroups/memory.txt](<https://lwn.net/Articles/548897/>) for details on how to use this feature. 

  * "Return probes" are now supported in user-space probing; they allow the activation of a breakpoint when a target function returns to its caller. 

  * The generation of POSIX timer IDs has changed; IDs are no longer guaranteed to be unique across the system. A process's timers can now be queried by reading `/proc/_PID_ /timers`. These changes make it possible for the checkpoint/restart feature to restore active timers without changing their IDs. 

  * POSIX and high-resolution timers support a new clock (`CLOCK_TAI`) which operates in [international atomic time](<https://en.wikipedia.org/wiki/International_Atomic_Time>). 

  * The `perf` command (along with the perf_events subsystem) can now do memory access profiling. 

  * The iSCSI subsystem has gained support for the [iSCSI extensions for RDMA (iSER)](<https://en.wikipedia.org/wiki/ISCSI_Extensions_for_RDMA>) protocol. 

  * CPU frequency governors can now exist in multiple instances with different tuning parameters. This feature is needed in heterogeneous multiprocessing systems (ARM big.LITTLE, for example) to allow different CPUs to run with different parameters. 

  * The ability to run scripts as executables (with the interpreter specified using the "`#!`" sequence) can now be built as a module — or left out altogether for systems that run no scripts. 

  * New hardware support includes: 

    * **Display and graphics** : Microsoft Hyper-V synthetic video devices and ILI Technology ILI9221/ILI9222 backlight controllers. 

    * **Hardware monitoring** : Analog Devices ADT7310/ADT7320 temperature monitors, Nuvoton NCT6779D hardware monitoring chips, National Semiconductor LM95234 temperature sensors, and ST-Ericsson AB8500 thermal monitors. 

    * **Input** : Apple infrared receivers. 

    * **Miscellaneous** : Advantech PCI-1724U analog output cards, Analog Devices AD7923 analog to digital interfaces, Qualcomm single-wire serial bus interfaces, Broadcom BCM2835 SPI controllers, Aeroflex Gaisler GRLIB SPI controllers, NVIDIA Tegra114 SPI controllers, Silicon Labs 5351A/B/C programmable clock generators, on-chip static RAM regions, TI TPS65090 battery chargers, Samsung EXYNOS5440 CPU frequency controllers, and ARM big.LITTLE CPU frequency controllers. 

    * **Networking** : Netlogic XLR/XLS network interfaces. 

    * **USB** : DesignWare USB2 USB controllers. 

    * **Video4Linux** : Rafael Micro R820T silicon tuners, ITE Tech IT913x silicon tuners, OmniVision OV7640 sensors, Philips UDA1342 audio codecs, Techwell TW9903, TW9906 and TW2804 video decoders, Silicon Laboratories 4761/64/68 AM/FM radios, Silicon Laboratories Si476x I2C FM radios, and Samsung EXYNOS4x12 FIMC-IS imaging subsystems. 

Note also that the Android "configurable composite gadget" driver has been removed from the staging tree. It is apparently difficult to maintain and no current hardware uses it. 

Changes visible to kernel developers include: 

  * The devtmpfs filesystem now provides drivers with the ability to specify which user and group ID should own a given device. This capability is somewhat controversial — there is resistance to encoding user and group ID policy in the kernel — but it will be useful for systems like Android. 

  * The staging tree has gained [a new "sync" driver](<https://lwn.net/Articles/540997/>) (from Android) that can be used for synchronization between other drivers. 

  * There is a new "dummy-irq" driver that does nothing other than register an interrupt handler. It exists to debug IRQ sharing problems by forcing the enabling of a specific interrupt line. 

  * A lot of the low-level USB PHY access functions have been changed to GPL-only exports. 

  * The new `idr_alloc_cyclic()` function allocates ID numbers in a cyclic fashion: when the given range is exhausted, allocations will start again at the beginning of that range. 

  * The workqueue subsystem has seen some substantial reworking which, among other things, should make it perform better on NUMA systems. There is also a new sysfs interface that can be used to tweak some workqueue parameters. 

If the usual pattern holds, this merge window should remain open until around May 12 and the 3.10 kernel can be expected in early July. As usual, LWN will follow the mainline as the merge window progresses.
