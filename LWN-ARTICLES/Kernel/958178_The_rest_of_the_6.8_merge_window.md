---
title: The rest of the 6.8 merge window
url: https://lwn.net/Articles/958178/
date: "January 22, 2024"
category: "Releases-6.8"
author: "By Jonathan Corbet January 22, 2024"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 22, 2024

Linus Torvalds was able to release [6.8-rc1](<https://lwn.net/ml/linux-kernel/CAHk-=wiB4iHTtfZKiy5pC24uOjun4fbj4kSX0=ZnGsOXadMf6g@mail.gmail.com/>) and close the 6.8 merge window on time despite losing power to his home for most of a week. He noted that this merge window is ""maybe a bit smaller than usual"", but 12,239 non-merge changesets found their way into the mainline, so it's not _that_ small. About 8,000 of those changes were merged since [the first-half summary](<https://lwn.net/Articles/957188/>) was written; the second half saw a lot of device-driver updates, but there were other interesting changes as well. 

Some of the most significant changes pulled in the latter half of the 6.8 merge window are: 

#### Architecture-specific

  * The riscv architecture has made [more information about supported ISA extensions](<https://git.kernel.org/linus/f352a28cc2fb>) on the current system available via the `riscv_hwprobe()` system call. See [`Documentation/arch/riscv/hwprobe.rst`](<https://www.kernel.org/doc/html/next/arch/riscv/hwprobe.html>) for details on what is available. 
  * Riscv can also now suspend to RAM if the SUSP SBI extension is present. 
  * Host-side support for [Intel Trust Domain Extensions (TDX)](<https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html>) has been merged; this will eventually allow KVM to create TDX-protected guests. [This documentation commit](<https://git.kernel.org/linus/4e1c7dddc717>) has some more information. 
  * The LoongArch architecture has added support for modules written in Rust. This architecture has also raised the minimum version of Clang that can be used to 18.0.0 — which has not been released yet. 

#### Core kernel

  * It is now possible to change the size of tracing sub-buffers used for the reporting of trace events to user space; see [this documentation commit](<https://git.kernel.org/linus/7c3f48026589>) for more information. 
  * One new "feature" — the scheduler performance regression encountered by Torvalds early in the merge window — has been removed with [this fix](<https://git.kernel.org/linus/e37617c8e53a>). 

#### Filesystems and block I/O

  * The ~~device-mapper~~ multi-device `MD_LINEAR`, `MD_MULTIPATH`, and `MD_FAULTY` targets have been [deprecated](<https://git.kernel.org/linus/608f52e30aae>) since the 5.14 release in 2021; they have now been removed. 

#### Hardware support

  * **Clock** : Qualcomm SC8280XP camera clock controllers, Qualcomm SM8650 global clock controllers, Qualcomm SM8650 TCSR clock controllers, Qualcomm SM8650 display clock controllers, Qualcomm SM8650 GPU clock controllers, Qualcomm QDU1000/QRU1000 ECPRI clock controllers, Qualcomm X1E80100 global clock controllers, MediaTek MT7988 clock controllers, Nuvoton MA35D1 realtime clocks, TI TPS6594 realtime clocks, and Analog Devices MAX31335 automotive realtime clocks. 
  * **GPIO and pin control** : Realtek DHC GPIO controllers, Nuvoton BMC NPCM7xx/NPCM8xx SGPIO controllers, Qualcomm SM8550 LPASS LPI pin controllers, Qualcomm SM8650, SM4450 and X1E80100 pin controllers, TI TPS6594 PMIC GPIO controllers, and Intel Meteor Point pin controllers. 
  * **Graphics** : Imagination Technologies PowerVR (Series 6 and later) and IMG GPUs, Synaptics R63353-based panels, and Ilitek ILI9805-based panels. **Also merged** is the Intel "Xe" driver for GPUs starting with the Tiger Lake generation. It is not enabled by default anywhere but that will change in some future kernel development cycle. 
  * **Hardware monitoring** : Monolithic Power Systems MP5990 hot-swap controllers, Monolithic Power Systems mp2856/mp2857 modulation controllers, Analog Devices LTC4286 and LTC4287 hot-swap controllers, and Gigabyte Waterforce X240/X280/X360 AIO CPU coolers. 
  * **Industrial I/O** : Maxim max34408/max344089 analog-to-digital converters, Bosch BMI323 I2C and SPI controllers, Microchip MCP9600 thermocouple EMF converters, Vishay VEML6075 UVA and UVB light sensors, Intersil ISL76682 light sensors, Melexis MLX90635 contactless infrared sensors, Honeywell HSC/SSC TruStability pressure sensors, Lite-On LTR-390UV-01 ambient light and UV sensors, Aosong AGS02MA TVOC sensors, Microchip MCP4801/02/11/12/21/22 digital-to-analog converters, and Analog Devices AD7091R8 analog-to-digital converters. 
  * **LED** : Allwinner A100 RGB LED controllers and Maxim 5970 indication LEDs. 
  * **Media** : Starfive camera subsystems, Chips&Media Wave codecs, GalaxyCore GC2145 and GC0308 sensors, THine THP7312 image signal processors, STMicroelectronics STM32 memory interface pixel processors, Techwell TW9900 video decoders, Allied Vision ALVIUM MIPI CSI-2 cameras, and OmniVision OV64A40 sensors. 
  * **Miscellaneous** : Apple SoC mailboxes, Qualcomm PMIC PDCharger ULOG providers, Microchip MCP2200 HID USB-to-GPIO bridges, Nintendo NSO controllers, AWS EC2 Nitro security modules, Intel visual sensing controllers, AMD AXI 1-wire bus host interfaces, Qualcomm SM8650, SM6115 and X1E80100 interconnects, MPS MP3309C backlight controllers, Adafruit Mini I2C gamepads, and Loongson LS2X APB DMA controllers. 
  * **Sound** : Qualcomm X1E80100 audio subsystems and Qualcomm WCD939x USBSS analog audio switches. 

#### Miscellaneous

  * The `perf` tool has gained support for [data-type profiling](<https://lwn.net/Articles/955709/>). Some more details, along with information on a the usual large pile of other `perf` changes, can be found in [this merge message](<https://git.kernel.org/linus/9d64bf433c53>). 

#### Security-related

  * See [this blog post](<https://paul-moore.com/blog/d/2024/01/linux_v68_merge_window.html>) from Paul Moore covering changes to the kernel's security subsystem in detail. 
  * The AppArmor security module has switched its policy-hash verification from the SHA-1 hash to SHA-256. 
  * The task of [removing the `strlcpy()` API](<https://lwn.net/Articles/905777/>) from the kernel is now complete. 

#### Virtualization and containers

  * The [guest-first memory feature](<https://lwn.net/Articles/949277/>) for KVM has been merged. Guest-first memory can be allocated for and mapped into KVM guests, but is inaccessible to the host, making it suited to confidential-computing applications. There is also a new `ioctl()` call where the expected attributes for guest memory (including a lack of mapping in the host) can be specified. [This changelog](<https://git.kernel.org/linus/6c370dc65374>) has some more information. 
  * KVM on arm64 systems has gained support for 52-bit (LPA2) physical addresses. 
  * KVM on x86 can now be built without Hyper-V emulation support, reducing the size of the resulting kernel. 

#### Internal kernel changes

  * The kernel now has [a `.editorconfig` file](<https://git.kernel.org/linus/5a602de99797>) that will automatically configure editors to the kernel's coding style. 
  * The new `check-uapi.sh` script can be used to detect inadvertent changes to the kernel's user-space API. See [`Documentation/dev-tools/checkuapi.rst`](<https://docs.kernel.org/next/dev-tools/checkuapi.html>) for details. 

If all goes according to plan (which it pretty much always does), the 6.8 kernel will be released on March 10 or 17\. Between now and then, though, there will certainly be a lot of bugs to find and fix.
