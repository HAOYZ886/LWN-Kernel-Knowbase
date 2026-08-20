---
title: The first half of the 6.10 merge window
url: https://lwn.net/Articles/973687/
date: "May 16, 2024"
category: "Releases-6.10"
author: "By Jonathan Corbet May 16, 2024"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 16, 2024

The merge window for the 6.10 kernel release opened on May 12; between then and the time of this writing, 6,819 non-merge commits were pulled into the mainline kernel for that release. Your editor has taken some time out from [LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>) in an attempt to keep up with the commit flood. Read on for an overview of the most significant changes that were pulled in the early part of the 6.10 merge window. 

#### Architecture-specific

  * Support for a number of early Alpha CPUs (EV5 and earlier) has been removed. As noted in [the merge message](<https://git.kernel.org/linus/736676f5c3ab>), these were the only machines supported by the kernel that did not provide byte-level memory access, and that created complications for support throughout the kernel. It is also the first non-x86 architecture to which the kernel was ported. Linus Torvalds amended the merge message to add: 

> I dearly loved alpha back in the days, but the lack of byte and word operations was a horrible mistake and made everything worse - including very much the crazy IO contortions that resulted from it. 
> 
> It certainly wasn't the only mistake in the architecture, but it's the first-order issue. 
> 
> So while it's a bit sad to see the support for my first alpha go away, if you want to run museum hardware, maybe you should use museum kernels. 

  * The [x32](<https://en.wikipedia.org/wiki/X32_ABI>) subarchitecture now supports shadow stacks. 
  * Arm64 systems have gained support for the [`userfaultfd()` write-protect feature](<https://lwn.net/Articles/940704/>). 
  * There is now a BPF just-in-time compiler for 32-bit ARCv2 processors. 

#### Core kernel

  * Rust abstractions for time handling within the kernel have been added. This work was [discussed](<https://lwn.net/Articles/924746/>) in early 2023 and has finally found its way in; see [this commit](<https://git.kernel.org/linus/48b7f4d29ac8>) for the current form of this interface. 
  * BPF programs now have the ability to use wait queues in the kernel; see [this merge message](<https://git.kernel.org/linus/6e10b6350a67>) for some more information. It is also now [possible](<https://git.kernel.org/linus/55d30cc90fd4>) for BPF programs to disable and enable preemption. 

#### Filesystems and block I/O

  * The new `F_DUPFD_QUERY` operation for `[fcntl()](<https://man7.org/linux/man-pages/man2/fcntl.2.html>)` allows a process to check whether two file descriptors refer to the same underlying file. This functionality is also provided by [`kcmp()`](<https://lwn.net/Articles/845448/>), but in a more restricted form that leaks less information from the kernel and, as a result, should be available even on systems where `kcmp()` is disabled. 
  * The block-throttling low-limit mechanism, [described](<https://elixir.bootlin.com/linux/v6.9/source/block/Kconfig#L122>) in the `Kconfig` file as ""a best effort limit to prioritize cgroups"", has been removed. It was marked as "experimental" since being introduced in 2017, does not appear to have acquired users, and complicated the maintenance of the block layer. 
  * The EROFS filesystem now supports Zstandard compression. 
  * The [dm-crypt](<https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/dm-crypt.html>) device-mapper target has a new `high_priority` option that allows it to use high-priority workqueues for its processing. This option can improve performance on larger systems, but defaults to "off" to avoid creating latency problems for other workloads (such as audio processing) on smaller systems. 

#### Hardware support

  * **GPIO and pin control** : pin controllers using the SCMI message protocol and Intel Granite Rapids-D vGPIO controllers. 
  * **Graphics** : Samsung S6E3FA7 panels, ARM Mali CSF-based GPUs, LG SW43408 panels, Raydium RM69380-based DSI panels, and Microchip LVDS serializers. 
  * **Hardware monitoring** : Analog Devices ADP1050 power-supply controllers, Lenovo ThinkStation EC sensors, and Infineon XDP710 hot-swap controllers. 
  * **Input** : WinWing Orion2 throttles. 

**Also:** the [BPF for HID drivers framework](<https://lwn.net/Articles/909109/>)2 is finally seeing some use with the addition of a number of small fixup programs to the kernel tree, [the first of which](<https://git.kernel.org/linus/04b3e5ab0555>) is for the XPPen Artist 24 device. Some [new udev functionality](<https://gitlab.freedesktop.org/libevdev/udev-hid-bpf>) is used to load these programs as needed. 
  * **Miscellaneous** : STMicroelectronics STM32 firewall-framework controllers, Arm Trusted Services secure partitions, NXP DCP key-storage devices, NVIDIA Tegra security engines, and Airoha SPI NAND flash interfaces. 
  * **Networking** : Airoha EN8811H 2.5 Gigabit PHYs, Realtek 8922AE PCI wireless network (Wi-Fi 7) adapters, Realtek 8723CS SDIO wireless network adapters, TI Gigabit PRU SR1.0 Ethernet adapters, Microsemi PD692x0 I2C power sourcing equipment controllers, TI TPS23881 I2C power sourcing equipment controllers, Renesas RZ/N1 Ethernet controllers, and Intel HCI PCIe Bluetooth controllers. 
  * **Sound** : Rockchip RK3308 audio codecs and Texas Instruments PCM6240 family audio chips. 

#### Miscellaneous

  * The version of the Rust language used with kernel code has been moved up to 1.78.0. Among other things, this change has made it possible to drop the kernel's forked version of the `alloc` crate, removing about 10,000 lines of code. A number of other changes have been made as well; see [this merge message](<https://git.kernel.org/linus/8f5b5f78113e>) and [this commit](<https://git.kernel.org/linus/56f64b370612>) for the full list. 

#### Networking

  * The performance of zero-copy send operations using io_uring has been significantly improved. It is also now possible to "bundle" multiple buffers for send and receive operations, again improving performance. 
  * The sending of file descriptors over Unix-domain sockets with `SCM_RIGHTS` messages has long been prone to the creation of reference-count cycles; see [this 2019 article](<https://lwn.net/Articles/779472/>) for one description of the problem and attempts to resolve it. The associated garbage-collection code has been massively reworked for 6.10, leading to a simpler and more robust solution; see [this merge message](<https://git.kernel.org/linus/da493dbb1f2a>) for some more information. 
  * There is now basic support for setting up [packet forwarding control protocol (PFCP)](<https://en.wikipedia.org/wiki/PFCP>) filters, though much of the work must be done in user space and only IPv4 is supported. 
  * TCP sockets now support the `SO_PEEK_OFF` socket option in the same way that Unix-domain sockets do. This allows the specification of an offset to be used when looking at data with `MSG_PEEK`. 

#### Security-related

  * The kernel now supports encrypted interactions with trusted platform module (TPM) devices; [this documentation commit](<https://git.kernel.org/linus/3d2daf9d592e>) has more information. 
  * The "crypto usage statistics" feature, which is seemingly unused, has been removed from the kernel. See [this commit](<https://git.kernel.org/linus/29ce50e078b8>) for a detailed justification for this removal. 
  * BPF programs now have access to the kernel's crypto framework. 

The 6.10 merge window can be expected to remain open until May 26\. Once it has closed, LWN will be back with a summary of what was pulled into the mainline for the latter part of this merge window.
