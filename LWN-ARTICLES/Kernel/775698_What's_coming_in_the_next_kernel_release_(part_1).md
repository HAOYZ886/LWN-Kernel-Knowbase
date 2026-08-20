---
title: "What's coming in the next kernel release (part 1)"
url: https://lwn.net/Articles/775698/
date: "December 31, 2018"
category: "Releases-5.0"
author: "By Jonathan Corbet December 31, 2018"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 31, 2018

When the [4.20 kernel](<https://lwn.net/Articles/775486/>) was released on December 23, Linus Torvalds indicated that he would try to keep to the normal merge window schedule despite the presence of the holidays in the middle of it. Thus far, he seems to be trying to live up to that; just over 8,700 changesets have been merged for the next release, which seems likely to be called 5.0. A number of long-awaited features are finally landing in the kernel with this release. 

Some of the more significant changes merged so far are: 

#### Architecture-specific

  * Intel's [processor trace](<https://lwn.net/Articles/648154/>) functionality is now supported for use by virtualized guests running under KVM. 
  * The arm64 architecture has gained support for a number of features including the [`kexec_file_load()`](<http://man7.org/linux/man-pages/man2/kexec_load.2.html>) system call, 52-bit virtual address support for user space, hotpluggable memory, per-thread stack canaries, and [pointer authentication](<https://lwn.net/Articles/718888/>) (for user space only at this point). [This commit](<https://git.kernel.org/linus/fbedc599e9b891a6756b1c9bc2eead02b02cce77>) has some documentation for the pointer-authentication feature. 

#### Core kernel

  * The [long-awaited](<https://lwn.net/Kernel/Index/#Scheduler-and_power_management>) energy-aware scheduling patches have finally found their way into the mainline. This code adds a new energy model that allows the scheduler to determine what the relative power cost of scheduling decisions will be. It will enable the mainline scheduler to get better results on mobile devices and, with luck, reduce or eliminate the scheduler patching that various vendors engage in now. 
  * 64-Bit versions of the `ppoll()`, `pselect6()`, `io_pgetevents()`, `recvmmsg()`, `futex()`, and `rt_sigtimedwait()` system calls have been added for 32-bit systems, making it possible to use these calls successfully after the year-2038 apocalypse. This completes the set of top-level system call conversions. [According to Arnd Bergmann](<https://git.kernel.org/linus/b12a9124eeb71d766a3e3eb594ebbb3fefc66902>): ""Hopefully in the next release we can wire up all 22 of those system calls on all 32-bit architectures, which gives us a baseline version for glibc to start using them"". 
  * The cpuset controller now works (with reduced features) under the version-2 control-group API. See the documentation updates in [this commit](<https://git.kernel.org/linus/4ec22e9c5a90e3809dd52014d5d239af8831a520>) for details. 

#### Filesystems and block layer

  * The Btrfs filesystem has regained the ability to host swap files, though with a lot of limitations (no copy-on-write, must be stored on a single device, and no compression allowed, for example). 
  * The [`fanotify()`](<http://man7.org/linux/man-pages/man7/fanotify.7.html>) mechanism supports a new `FAN_OPEN_EXEC` request to receive notifications when a file is opened to be executed. 
  * The legacy (non-multiqueue) block layer code has been removed, now that no drivers require it. The legacy I/O schedulers (including CFQ and deadline) have been removed as well. 
  * "Binderfs" is a new virtual filesystem used to control the Android binder subsystem. See [this commit](<https://git.kernel.org/linus/3ad20fe393b31025bebfc2d76964561f65df48aa>) for some information. 

#### Hardware support

  * **Audio** : AKM AK4118 S/PDIF transceivers, Amlogic AXG SPDIF inputs, Xilinx I2S audio interfaces, and Cirrus Logic CS47L35/85/90/91 and WM1840 codecs. 
  * **Graphics** : Olimex LCD-OLinuXino bridge panels, Samsung S6D16D0 DSI video mode panels, Truly NT35597 WQXGA dual DSI video mode panels, and Himax HX8357D LCD controllers. 
  * **I3C** : The kernel has a new subsystem for [I3C](<https://mipi.org/specifications/i3c-sensor-specification>) devices, along with drivers for Cadence and Synopsys DesignWare controllers. 
  * **Industrial I/O** : Analog Devices AD7949 and AD7124 analog-to-digital converters, Texas Instruments DAC7311, DAC6311, and DAC5311 digital-to-analog converters, Vishay VCNL4035 light and proximity sensors, PNI RM3100 3-Axis magnetometers, and Microchip MCP41xxx/MCP42xxx digital potentiometers. 
  * **Media** : Sony IMX214 sensors, SECO Boards HDMI CEC interfaces, Allwinner V3s camera sensor interfaces, Rockchip VPU JPEG encoders, Aspeed AST2400 and AST2500 video engines, and Intel ipu3-imgu image processing units. 
  * **Miscellaneous** : Microchip MCP16502 power-management ICs, Macronix MX25F0A SPI controllers, Nuvoton NPCM peripheral SPI controllers, Cavium ThunderX2 SoC uncore PMUs, Alcor Micro AU6601 and AU6621 SD/MMC controllers, TI AM654 SDHCI controllers, Cadence GPIO controllers, Microchip SAMA5D1 PIOBU GPIO controllers, Spreadtrum SC27XX fuel gauges, and Intel Stratix10 SoC FPGA managers. 
  * **Networking** : Aquantia AQtion 5/2.5GbE USB network interfaces, Quantenna QSR1000/QSR2000 wireless network interfaces, and MediaTek GMAC Ethernet controllers. 
  * **USB** : Cadence Sierra USB PHYs and Freescale i.MX8M USB3 PHYs. 

#### Networking

  * Generic receive offload (GRO) can now be enabled on plain UDP sockets. If the numbers in [this commit](<https://git.kernel.org/linus/e20cf8d3f1f763ad28a9cb3b41305b8a8a42653e>) are to be believed, the result is a significant increase in receive bandwidth and a large reduction in the number of system calls required. 
  * ICMP error handling for UDP tunnels is now supported. 
  * The [`MSG_ZEROCOPY`](<https://lwn.net/Articles/726917/>) option is now supported for UDP sockets. 

#### Security

  * Support for the [Streebog hash function](<https://tools.ietf.org/html/rfc6986>) (also known as GOST R 34.11-2012) has been added to the cryptographic subsystem. 
  * A new crypto mode called "Adiantum" has been added as a replacement for [the (removed) Speck algorithm](<https://lwn.net/Articles/761992/>). Adiantum is intended to be secure while being fast enough to perform disk encryption on low-end handsets; see [this commit message](<https://git.kernel.org/linus/059c2a4d8e164dccc3078e49e7f286023b019a98>) for details. As part of this work, support for the XChaCha12 and XChaCha20 stream ciphers was also added. 
  * The kernel is now able to support non-volatile memory arrays with built-in security features; see [`Documentation/nvdimm/security.txt`](<https://lwn.net/Articles/775740/>) for details. 

#### Internal kernel changes

  * There is a new "software node" concept that is meant to be analogous to the "firmware nodes" created in ACPI or device-tree descriptions. See [this commit](<https://git.kernel.org/linus/59abd83672f70cac4b6bf9b237506c5bc6837606>) for some additional information. 
  * The first two of the retpoline-elimination mechanisms described in [this article](<https://lwn.net/Articles/774743/>) have been merged. improving performance in core parts of the DMA mapping and networking layers. 
  * The [software-tag-based mode for KASAN](<https://lwn.net/Articles/766768/>) has been added for the arm64 architecture. 
  * The switch to using [JSON schemas for device-tree bindings](<https://lwn.net/Articles/771621/>) has begun with the merging of the core infrastructure and the conversion of a number of binding files. 
  * The long-deprecated `SUBDIRS=` build option is finally going away in the 5.3 merge window; users will start seeing a warning as of 5.0. The `M=` option should be used instead. 

Before the 4.20 release, Torvalds had suggested that this merge window would go for longer than usual given the presence of the holidays in the middle. The pace of merging so far suggests that this plan has fallen by the wayside, though, and maintainers should not count on the merge window being open past January 6\. As always, LWN will follow up with a summary of the changes that are merged between now and the closing of the merge window, whenever that may be.
