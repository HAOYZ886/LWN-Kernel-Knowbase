---
title: The second half of the 4.17 merge window
url: https://lwn.net/Articles/751482/
date: "April 16, 2018"
category: "Releases-4.17"
author: "By Jonathan Corbet April 16, 2018"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 16, 2018

By the time the 4.17 merge window was closed and [4.17-rc1](<https://lwn.net/Articles/751885/>) was released, 11,769 non-merge changesets had been pulled into the mainline repository. 4.17 thus looks to be a typically busy development cycle, with a merge window only slightly more busy than 4.16 had. Some 6,000 of those changes were pulled after [last week's summary](<https://lwn.net/Articles/750928/>) was written. There was a lot of the usual maintenance work in those patches (over 10% of those changes were to device-tree files, for example), but also some more significant changes, including: 

#### Core kernel

  * The `CLOCK_MONOTONIC` and `CLOCK_BOOTTIME` clocks used to differ only in that the latter is fast-forwarded after a suspend-and-resume cycle. As of 4.17, `CLOCK_MONOTONIC` is also moved forward to reflect the time that the system spent suspended. As a result, the two timers are now identical and have been unified within the kernel. Among other things, that change eliminates a potentially surprising behavior wherein the offset between the monotonic and realtime clocks would change after a resume. Thomas Gleixner [noted](<https://git.kernel.org/linus/d6ed449afdb38f89a7b38ec50e367559e1b8f71f>): ""There might be side effects in applications, which rely on the (unfortunately) well documented behaviour of the MONOTONIC clock, but the downsides of the existing behaviour are probably worse."" 

If applications do break, this change may have to be reverted. Meanwhile, there is a new clock (`CLOCK_MONOTONIC_ACTIVE`) that only advances when the system is actually running. 
  * The new `INOTIFY_IOC_SETNEXTWD` `ioctl()` command allows inotify users to specify the number of the descriptor they would like to see returned for the next watch descriptor they create. This is used for checkpoint/restart. 
  * After a few years of waiting, the [histogram trigger](<https://lwn.net/Articles/635522/>) feature was added to the tracing subsystem. This mechanism enables the easy creation, in kernel space, of histograms from tracing data. 
  * The `mmap()` system call supports a new `MAP_FIXED_NOREPLACE` option. Like `MAP_FIXED`, it tries to place the new memory region at a user-supplied address. Unlike `MAP_FIXED`, though, it will not replace an existing mapping at that address; instead, it will fail with `EEXIST` if such a mapping exists. This is the change that was discussed last year as [`MAP_FIXED_SAFE`](<https://lwn.net/Articles/741369/>); it seems that the battle over the proper name for the feature has finally been resolved. 

#### Architecture-specific

  * The ARM architecture has gained support for the "system control and management interface", or SCMI. It is a set of standards for system management and, in particular power management. 
  * 64-Bit PowerPC systems now have the ability to address up to 4PB of memory. 
  * Support for POWER4 processors was accidentally ([they swear](<https://git.kernel.org/linus/49a695ba723224875df50e327bd7b0b65dd9a56b>)) broken in 2016, and nobody complained. So support for those processors has been removed entirely on the assumption that nobody is using them anymore. 

#### Filesystems

  * The overlayfs filesystem can, at times, present different inode numbers for the same file at different times, potentially confusing applications that use those numbers. The "`xino`" option added for 4.17 will store the filesystem ID in the upper part of the inode number, which allows it to present inode numbers that will not change over time. Some information can be found in [Documentation/filesystems/overlayfs.txt](<https://lwn.net/Articles/751908/>). 

#### Security-related

  * The kernel now supports the Speck cipher, a block cipher that is said to outperform AES on systems without hardware AES support. 
  * AES encryption in Cipher Feedback Mode is now supported; this is required for TPM2 cryptography. 
  * The SM4 symmetric cipher algorithm is supported; it is ""an authorized cryptographic algorithm for use within China"" according to [commit](<https://git.kernel.org/linus/747c8ce4e710cf2d72d115f84b2d0d6f4aa504b4>). 
  * The SCTP protocol now has complete SELinux support; see [Documentation/security/SELinux-sctp.rst](<https://lwn.net/Articles/751656/>) for details. 
  * The AppArmor security module has gained basic support for the control of socket use. See [this commit](<https://git.kernel.org/linus/56974a6fcfef69ee0825bd66ed13e92070ac5224>) for a little bit of documentation. 

#### Hardware support

  * **Audio** : Texas Instruments PCM1789 codecs, AKM AK4458 and AK5558 codecs, Rohm BD28623 codecs, Motorola CPCAP codecs, Maxim MAX9759 speaker amplifiers, ST TDA7419 audio processors, and UniPhier AIO audio subsystems, 
  * **Cryptographic** : ARM TrustZone CryptoCell security processors and TI Keystone NETCP SA hardware random-number generators. 
  * **Industrial I/O** : Melexis MLX90632 infrared sensors, Analog Devices AD5272 digital potentiometers, On Semiconductor LV0104CS ambient light sensors, and Microchip MCP4017/18/19 digital potentiometers. 
  * **USB** : HiSilicon STB SoCs COMB PHYs, AMLogic Meson GXL and GXM USB3 PHYs, STMicroelectronics STM32 USB HS PHY controllers, HiSilicon INNO USB2 PHYs, Motorola Mapphone MDM6600 USB PHYs, Pericom PI3USB30532 Type-C cross switches, ELAN USB touchpads, and devices supporting USB class 3 audio. 
  * **Miscellaneous** : QCOM on-chip GENI based serial ports, MediaTek SoC gigabit Ethernet controllers, Raspberry Pi 3 GPIO expanders, Nintendo Wii GPIO controllers, Spreadtrum SC9860 platform GPIO controllers, RAVE SP power buttons, PhoenixRC flight controller adapters, HiSilicon hi3660 mailbox controllers, Socionext SynQuacer I2C controllers, Intersil ISL12026 realtime clocks, Nuvoton NPCM750 watchdog timers, Mediatek MT2701 audsys clocks, Allwinner H6 clock controllers, Silicon Labs 544 I2C clock generators, Synopsys DesignWare AXI DMA controllers, and MediaTek High-Speed DMA controllers. 

#### Other

  * The ABI for 32-bit RDMA users has changed in incompatible ways. The changes are [justified](<https://git.kernel.org/linus/19fd08b85bc7e0502b55cd726f466df82ee7e777>) with the claim that there are no actual users of the 32-bit mode now, but some may be coming in the future. 

#### Internal kernel changes

  * The way that system calls are invoked on the x86-64 architecture has been reworked to make it more uniform and flexible. The new scheme has also been designed to prevent unused (but caller-controlled) data from getting onto the call stack — where it could perhaps be used in a speculative-execution attack. 
  * The lexer and parser modules used by the kernel build process are now themselves built on the target system (requiring `flex` and `bison`) rather than being shipped in the kernel repository. 

As expected, the final diffstat for this merge window shows that more lines of code were deleted than added — 191,000 more. This is only the third time in the kernel's history that a release has been smaller than its predecessor. 

Also possibly worthy of note is that the [final SCSI pull](<https://git.kernel.org/linus/f0d98d85831bf1a3b1f56f8c14af60797aaca536>) pushed the kernel repository to over six-million objects. Linus added: ""I was joking around that that's when I should switch to 5.0, because 3.0 happened at the 2M mark, and 4.0 happened at 4M objects. But probably not, even if numerology is about as good a reason as any."" 

This kernel now enters the stabilization process, which will culminate in the final 4.17 (or maybe 5.0?) release in early June.
