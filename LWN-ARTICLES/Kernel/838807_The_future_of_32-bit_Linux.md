---
title: "The future of 32-bit Linux"
url: https://lwn.net/Articles/838807/
date: "December 4, 2020"
category: "Architectures-32-bit systems"
author: "December 4, 2020 This article was contributed by Arnd Bergmann"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

December 4, 2020

This article was contributed by Arnd Bergmann

The news for processors and system-on-chip (SoC) products these days is all about 64-bit cores powering the latest computers and smartphones, so it's easy to be misled into thinking that all 32-bit technology is obsolete. That quickly leads to the idea of removing support for 32-bit hardware, which would clearly make life easier for kernel developers in a number of ways. At the same time, a majority of embedded systems shipped today do use 32-bit processors, so a valid question is if this will ever change, or if 32-bit will continue to be the best choice for devices that do not require significant resources. 

To find an answer, it is worth taking a look at different types of systems supported in Linux today, how they have evolved over time with the introduction of 64-bit processors, why they remain popular, and what challenges these face today and in the future.

#### 32-Bit desktops

Linux was first written as a desktop system for IBM PC compatibles, and was eventually ported to almost every other desktop platform available in the 1990s, including a lot of the early Unix workstations across all architectures. Over time, all of these were replaced with 64-bit machines or they disappeared from the market, or in case of Unix workstations, both. The earliest i386, Arm, MIPS, and PowerPC processors all got phased out over time, but a lot of others still remain. 

The table shows the 32-bit desktop platforms that proved popular enough to make it into mainline Linux and stay there, supported mainly by loyal hobbyists: 

> Platform | Architectures | Earliest supported machine | Last supported 32-bit machine | End of marketing | Replaced by  
> ---|---|---|---|---|---  
> Unix workstation |  m68k   
>  MIPS  
> SPARC   
>  PA-RISC   
>  PowerPC |  Sun-3/60   
>  (1985)  |  IBM RS/6000 43P-240,   
>  HP Visualize B180L   
>  (1997) |  1999 |  x86 Linux, 64-bit Unix  
> Apple Macintosh |  m68k  
> PowerPC  
> ia32 |  Mac II (1987) |  Macbook  
> (2006) |  2007 |  x86-64, arm64 Mac  
> IBM compatible PC |  ia32 |  PS/2 model 70 486   
>  (1989) |  Intel Atom D270 Netbooks  
> (2009) |  2011 |  x86-64 PC  
> Amiga |  m68k   
>  PowerPC |  A2500   
>  (1989) |  AmigaOne SE  
> (2003) |  2004 |  -  
> Atari |  m68k |  Atari TT  
> (1990) |  Falcon  
> (1992) |  1993 |  -  
> Acorn RiscPC |  Arm |  StrongARM RiscPC   
>  (1996) |  StrongARM RiscPC J233   
>  (1997) |  2003 |  -  
> Netwinder |  Arm   
>  x86 |  Netwinder "Frog" (1999) |  NetWinder 3100  
>  (2001) |  2001 |  -  
> Android Tablet/Laptop |  Arm   
>  ia32  
> MIPS |  Nexus 7  
> (2012) |  Various, Android 8  
> (2019) |  Ongoing |  arm64  
> ChromeOS |  Arm |  Samsung 303c   
>  (2012) |  Asus Chromebook Flip   
>  (2015) |  2017 |  x86-64, arm64  
> Baikal T1 |  MIPS |  Tavolga Terminal   
>  (2018) |  Tavolga Terminal   
>  (2018) |  Ongoing |  arm64  
  
Android tablets using Armv7 processors are the only platform from the list above that remains widely available in 2020, but the end of marketing is in sight here as well. Tablets based on 64-bit Armv8 CPUs are not just more powerful but often also cheaper. Mainline support for many 32-bit Android tablets is only starting to get added to the kernel. (For those who are unfamiliar with the taxonomy of Arm processor variants, [this table](<https://en.wikipedia.org/wiki/List_of_ARM_microarchitectures>) describes them all). 

The Baikal T1 chip is a curiosity, as it only exists for the Russian domestic market to avoid relying on chip imports. It too has an Armv8 successor, but that is not yet widely deployed. Kernel support for the Baikal T1 was added in 2020.

#### Traditional embedded Linux

The use of Linux in embedded devices started in the late 1990s and, over time, spread to around 30 architectures, largely replacing other embedded operating systems. A lot of these were custom architectures made by companies that build system-on-chip designs around their own CPU cores. Over time, licensable CPU IP cores took over, with MIPS, PowerPC, x86, and SuperH cores being popular until Arm started replacing them. 

These are the most recent licensable 32-bit generations for each of the main processor families supported in Linux-5.10: 

> Core |  Architecture |  Introduced |  Replacement |  New Architecture  
> ---|---|---|---|---  
> Renesas SH-4A |  SuperH |  2004 |  (Licensed from Arm) |  Armv7-A, Armv8-A  
> AMCC PowerPC 460 |  Power ISA 2.03 |  2006 |  X-Gene |  Armv8-A  
> IBM PowerPC 470 |  Power ISA 2.04 |  2008 |  IBM A2 |  ppc64le  
> Freescale e500mc |  Power ISA 2.06 |  2008 |  (Licensed from Arm) |  Armv7-A, Armv8-A  
> MIPS Warrior P5600 |  MIPS32r5 |  2014 |  P6600, I8500 |  MIPS64r6, RV64  
> Arm Cortex-A17 |  Armv7-A |  2014 |  (various) |  Armv8-A  
> Cadence Xtensa LX7 |  Xtensa LX |  2016 |  (unknown) |  (unknown)  
> Andes D15F |  NDS32v3 |  2017 |  A25, AX25 |  RV32, RV64  
> OpenCores mor1kx v5 |  OpenRISC 1000 |  2017 |  (various) |  RV32, RV64  
> Microblaze v10.0 |  Microblaze32 |  2017 |  Microblaze v11.0 |  Microblaze32,Microblaze64  
> ARC HS4x |  ARCv2 |  2017 |  HS5x, HS6x |  ARCv3 (32/64 bit)  
> MIPS Warrior I7200 |  nanoMIPS |  2017 |  I8100 |  RV64  
> T-HEAD CK870 |  C-SKY v2 |  2018 |  C910 |  RV64  
> Gaisler LEON5 |  SPARCv8 |  2019 |  NOEL-V |  RV64  
  
As the table shows, almost all of these cores have introduced 64-bit replacements from their original designers, but most manufacturers keep on supporting the 32-bit versions as well. Hitachi (now Renesas) and Freescale (now NXP) introduced 64-bit core designs of their SuperH and PowerPC cores, but subsequently abandoned them in favor of licensed Arm Cortex-A cores. 

The OpenRISC project is still active, but a lot of its contributors have moved on to working on RISC-V designs. Cadence Tensilica has not announced a replacement for the Xtensa LX7 but is apparently focusing on its DSP cores. 

Some CPU cores end up being used for a long time in new SoCs. For instance, the Arm9 CPU was originally created in 1998, with the Arm926EJ-S update from 2001 still making it into the new Microchip SAM9X60 in 2019. Not only that, Microchip also still supports mainline Linux on its ancestor, the 2003 vintage AT91RM9200, and promises to sell it as long as there is demand. 

The manufacturers' specifications for product longevity of a part can give some indication of how long it will be used, but this is not always the same as the time for which it will receive kernel upgrades. Many embedded systems are shipped with an old kernel and never upgraded, others may be required to get periodic upgrades many years after the last one has shipped. 

These are some more examples of notable chips that are built on older cores or CPU architectures but will remain in production for several years to come, usually side by side with more powerful and newer Armv7 or Armv8 based parts. 

> SoC name |  CPU |  Introduced |  Armv7/v8 Replacement |  Projected EOL  
> ---|---|---|---|---  
> Renesas SH7734 |  SH-4A |  2010 |  RZ, R-Car series |  March 2025  
> NXP QorIQ P3041 |  PowerPC e500mc |  2010 |  QorIQ Layerscape |  June 2025  
> NXP MCF5441x |  Coldfire v4 (m68k) |  2010 |  i.MX |  September 2025  
> Broadcom BCM2835 |  Arm1176JZF-S |  2011 |  BCM2836+ |  January 2026  
> Qualcomm QCA9531 |  MIPS 24Kc |  2014 |  IPQ series |   
> Intel Quark SE C1000 |  Quark (486-like) |  2015 |  - |  July 2022  
> Allwinner F1C200s |  Arm926EJ-S |  2017 |  sunxi series |   
> Microchip SAM9X60 |  Arm926EJ-S |  2019 |  SAMA5 |   
> DM&P Vortex86EX2 |  Dual i686-like |  2019 |  - |  February 2029  
> Mediatek MT7621DAT |  MIPS 1004Kc |  2019 |  MT7622 |   
> Ingenic X2000 |  Xburst2 (MIPS32r5) |  2020 |  - |   
  
The end-of-life date in this table usually refers to the manufacturer's last guaranteed shipment date, often 10 or 15 years after the introduction, but for popular chips like the BCM2835, kernel updates may be needed long after the last chip has been sold. The m68k-based Coldfire MCF5441x is interesting because it represents the oldest architecture that Linux runs on, and it will still be sold until at least 2025, which is 40 years after the introduction of its Sun-3 ancestor, the oldest machine with Linux support. At that point it will have outlived almost every other 32-bit architecture that was meant to replace it. 

The more recent examples tend to be built as system-in-package processors that include 32 to 256 megabytes of DRAM. This helps further reduce board design cost, which is the main reason to use any of these parts instead of the more common Armv7 alternatives in new designs.

#### Microcontrollers and DSPs

The line between an SoC and a microcontroller can be hard to draw. Typically a microcontroller is a highly integrated system that leaves out one or more of the essential parts of a modern SoC: a memory-management unit (MMU), data cache, external DRAM memory, or sometimes an IEEE-754-compatible floating-point unit. Linux does not strictly require any of those, but running without an MMU in particular limits the applications that can be used on the system. 

Digital signal processors (DSPs) are another special class of processor. Like the now-removed Blackfin architecture, both the Texas Instruments C6x and Qualcomm Hexagon DSPs are capable of running Linux in a limited way, but this rarely seems to happen. Typically, these processors instead run bare-metal code and communicate with Linux running on an Arm core on the same chip. 

Support for MMU-less processors was merged into Linux-2.5.45 in 2002 and has been proclaimed to be dying ever since. At its height, 12 MMU-less CPU architectures were supported, but most of those are obsolete and no longer work with mainline kernels. The following are some of the still-supported SoCs as of Linux-5.10: 

> MCU name |  CPU |  Introduced  
> ---|---|---  
> NXP MCF537x |  Coldfire v3 |  2007  
> NXP LPC4357 |  Cortex-M4 |  2012  
> TI TMS320C6678 |  C66 |  2014  
> J-core FPGA SoC |  J2 |  2016  
> ST STM32H7 |  Cortex-M7 |  2016  
> Mediatek MT3620 |  Cortex-A7 |  2017  
> NXP i.MX RT1050 |  Cortex-M7 |  2017  
> Kendryte K210 |  RV64 |  2018  
> Espressif ESP32-S2 |  Xtensa LX7 |  2019  
  
The most active microcontroller platform running Linux is now the STM32 series. These can be even cheaper than the low-end MIPS32 or Armv5 chips, so they are sometimes used when manufacturing cost is more important than performance or the additional development effort. Power consumption and physical size can also be reasons to use them. Some products have shipped running inside the built-in 2MB of SRAM, others use external DRAM or [PSRAM](<https://en.wikipedia.org/wiki/Dynamic_random-access_memory#PSRAM>), which again increases cost. 

Kernel support for the i.MX RT series, MT3620, and ESP32 exists out of tree but has not been submitted for inclusion in mainline Linux. The Kendryte K210 is special for being a 64-bit MMU-less CPU system; the kernel port mainly exists because there are still no other low-cost RISC-V chips with Linux support. The Mediatek MT3620 is another special case, it is designed for the Microsoft Azure Sphere and does have an MMU, but runs Linux in only 4MB of integrated [SRAM](<https://en.wikipedia.org/wiki/Static_random-access_memory>). 

Linux on microcontrollers competes with a wide range of open-source realtime operating systems including [Zephyr](<https://www.zephyrproject.org/>), [NuttX](<https://nuttx.apache.org/>), [Mbed](<https://os.mbed.com/>), and [FreeRTOS](<https://www.freertos.org/>). These tend to need less memory and are better suited for running without a memory-management unit.

#### Linux on Armv7-A

It's hard to overstate the impact that Armv7 has had on embedded computing. This can be largely attributed to the never-ending demand for faster and more efficient processors in smartphones. Over the evolution from the high-end Nokia N96 in 2008 to this year's Apple iPhone 12, we see a 10x increase in clock frequency, an 8x increase in pipeline width, and a 6x increase in the number of cores. 

The Cortex-A9 managed to surpass practically all competing 32-bit cores of the time. It combined the advantages of an out-of-order pipeline, up to four cores, NEON SIMD, and 40nm manufacturing, and went on to become the most successful licensable 32-bit CPU core of all time. In Linux-5.10, there are 452 devicetree files based on 76 SoCs, or 30 SoC families, far more than any other Arm core, and more than all non-Arm machines combined. 

While the popular Cortex-A9 was the most advanced core in the first generation of Armv7-A, which includes Cortex-A8, Cortex-A5, PJ4, and Scorpion, the second generation Armv7VE cores ended up with the opposite situation. The high-end Cortex-A15, Cortex-A17, Brahma-B15, and Krait cores made a big entrance in mobile phones, automotive, and networking equipment from 2012 to 2013, but as they ran into limitations of the 32-bit virtual address space, they were quickly replaced with the emerging 64-bit Armv8 based products. 

Cortex-A7, in comparison, is a relatively simple, in-order CPU core, but on the typical 28nm process it beats a 40nm Cortex-A9 in performance and outclasses it in power consumption and die area. It also offers many of the features of the Armv8 cores that were integrated into Armv7VE: virtualization, large physical address extensions (LPAE), up to eight cores, big.LITTLE configurations, and hardware integer division. Seven years after its launch, Cortex-A7 remains the most common option for new low-cost embedded SoCs with two or four cores, most commonly paired with around 256MB of DDR3L memory per core either inside a system-in-package or on the board. 

Cortex-A9 and Cortex-A7 together with their 64-bit counterparts have replaced most other 32-bit architectures in embedded systems, including the MIPS32r2 cores that were common in wireless networking SoCs until about 2017. The 64-bit Cortex-A53 is now used in more SoCs than the Cortex-A7, but is still behind the Cortex-A9. 

There are two obvious factors that will determine the eventual demise of Armv7-A:

  * A system that needs more than 2GB of memory or higher performance than a quad-core Cortex-A7 ends up on 64-bit hardware.
  * Cortex-A7 is not available below 28nm semiconductor technology and loses its cost advantage over 64-bit cores as 22nm and 14nm processes get cheaper.

The Cortex-A32 core is a 32-bit version of the 64-bit Cortex-A35 that could theoretically address both the performance and the manufacturing points, but so far no SoC has been built around it. Apparently there is simply no point in using a feature-reduced core when the Cortex-A35 can do all the same things but also support 64-bit code at only a marginally higher cost. This means that, unless a new 32-bit Arm or RISC-V core manages to take over the extreme low end, Cortex-A7 will be the end of the line for 32-bit Linux. 

> ![\[Armv7 core use over time\]](https://static.lwn.net/images/2020/arnd-soc-graph.png)

This graph shows how many SoCs use each of the Armv7 CPU cores over time as the kernel adds devicetree source files for each one. Cortex-A9 initially saw the highest growth but has stagnated since, along with all cores other than the Cortex-A7. This timeline is delayed from the introduction of the hardware because support for older SoCs is sometimes added many years after the hardware was built. Looking at individual boards, there would be an additional delay based on how long an SoC is used in new products.

#### Year 2038

It is well understood that many 32-bit systems will stop working in the year 2038 when the 32-bit `time_t` overflows. Extrapolating from the current popularity of Armv7 SoCs and the time that chips for older architectures are getting deployed, it seems likely that 32-bit hardware running Linux will be sold well into the 2030s and in some cases stay in use for decades after a product has first shipped, so this is a very real threat. 

After years of work, the kernel itself is now completely converted to using 64-bit `time_t` internally, and the musl C library has switched over, but any distributions based on the GNU C Library (glibc) are still affected by this problem and need to be rebuilt against a future version of the library once its 64-bit time support has been completed. 

The degree to which distributions will be rebuilt is an open question, given the declining interest in 32-bit architectures among developers. Many binary distributions, like Fedora, Ubuntu, and openSUSE Leap, have dropped support for all 32-bit architectures other than Armv7 and are likely to drop that as well before they would consider rebuilding against a new glibc. 

Debian still supports its official ports to Armv5 (armel), Armv7 (armhf), x86-32 (i386), and MIPS32r2 (mipsel), as well as non-official 32-bit ports to PowerPC, SuperH, PA-RISC, Motorola 68000, and x32. The port to Armv7 is likely to outlive all the other architectures the way it did on the other distributions, possibly for another ten years after the remaining ones are gone. This means there will likely be embedded users that want Debian armhf to work beyond 2038. 

While there was [a plan to move to a time64 glibc for Debian 11 (Bullseye) on armhf](<https://lwn.net/Articles/812767/>), the earliest possible release is now Debian 12 (Bookworm). There are still open questions regarding whether this should replace the existing armhf port or whether it needs to be able to coexist as a multiarch target with a new name and target triplet, which adds more work. These questions may become moot if there is not enough interest among porters to keep Debian supported on 32-bit architectures in the future, which does seem to be a growing problem. 

In embedded systems that compile everything from source using the [Yocto Project](<https://www.yoctoproject.org/>), [Buildroot](<https://buildroot.org/>), [OpenWRT](<https://openwrt.org/>), or similar distributions, there is usually already an option to use the musl-1.2 library to get full support for 64-bit `time_t`, if this isn't already the default. 

#### Highmem

When a 32-bit system has more physical RAM than it can map into its linear virtual address space, anything outside of that linear mapping is called highmem and requires using the `kmap()`, `kmap_atomic()`, or the new `kmap_thread()` helpers to be accessible temporarily from kernel code. Requiring all user-space memory access to be written based on these interfaces is a burden to not just 32-bit architectures but also 64-bit ones. Given the observation that 32-bit systems with a lot of memory are actually getting rarer over time, the question of [deprecating highmem](<https://lwn.net/Articles/813201/>) comes up on [a regular basis](<https://lwn.net/Articles/836144/>). 

This is a list of common 32-bit systems with large memory requirements, to show which machines would be impacted the most from deprecating highmem: 

> Platform |  Maximum memory |  Introduced |  Processor |  Typical application  
> ---|---|---|---|---  
> Intel Xeon MP |  64 GB |  2002 |  Netburst (x86) |  Server  
> Calxeda Midway |  16 GB |  2013 |  Cortex-A15 |  Server  
> HiSilicon HiP04 |  16 GB |  2014 |  Cortex-A15 |  Server  
> Annapurna AL5140 |  8 GB |  2014 |  Cortex-A15 |  NAS Server  
> NXP QorIQ P4080 |  16 GB |  2008 |  e500mp (ppc32) |  Networking  
> Intel Axxia 5500 |  8 GB |  2013 |  Cortex-A15 |  Networking  
> TI Keystone 2 |  8 GB |  2013 |  Cortex-A15 |  Networking  
> Marvell Armada XP |  8 GB |  2012 |  PJ4B-MP |  Networking  
> Nvidia Tegra K1 (T124) |  8 GB |  2014 |  Cortex-A15 |  Industrial  
> Renesas RZ/G1H |  8 GB |  2016 |  Cortex-A15 |  Industrial  
> NXP i.MX 6QuadPlus |  4 GB |  2015 |  Cortex-A9 |  Industrial  
> Baikal T1 |  8 GB |  2015 |  MIPS P5600 |  PC  
> Rockchips RK3288 |  4 GB |  2014 |  Cortex-A17 |  Chromebook  
> Samsung Exynos 5800 |  4 GB |  2013 |  Cortex-A15 |  Chromebook  
> Intel Core Duo |  3 GB |  2006 |  P6 (x86) |  Laptop  
> NXP MPC7448 |  2 GB |  2005 |  PowerPC G4 |  Laptop  
  
The largest configurations appear in large server systems that were replaced with larger, 64-bit server systems many years ago. The same is true for most 32-bit PC and laptop systems with 2GB of memory or more. Early Chromebooks used Cortex-A15 or A17 CPU cores with 4GB of memory; they have stopped getting software updates in June 2020 but are often still in active use and can run modern Armv7 Linux distributions as a long-term alternative. 

For embedded industrial and networking systems, 32-bit configurations with up to 8GB exist and still require future kernel updates, but they are rare as the practical configuration is limited by the available DDR3 memory they use. A board with 8GB of DDR3 memory needs eight 8Gb chips that are expensive and not widely available, or sixteen 4Gb chips in a multi-rank setup and complex board layout. Later 64-bit SoCs in comparison often use LP-DDR4 memory that supports up to 8GB with a single 64Gb chip. 

Configurations with 2GB or 4GB of DDR3 or LP-DDR3 memory are common enough on Cortex-A15-based embedded systems that they will have to be supported for a long time. Cheaper Cortex-A7 based SoCs, instead, have a single narrow DDR3 interface, which limits the practical configurations to one or two chips with either 2Gb or 4Gb each, and avoids the use of highmem when all memory fits into a 1GB low-memory area. LP-DDR4 or other technologies that allow higher capacity are rarely used on these cheapest SoCs, because they tend to be more expensive for small configurations. 

A rough plan for removing highmem, based these observations, could unfold like this: 

  * Support for highmem gets worse over time, as the kernel is modified to work better on 64-bit systems while adding larger overhead on 32-bit kernels. This can be seen happening with the `kmap_thread()` addition.
  * 32-Bit machines with the largest memory configurations, including everything over 4GB, are rare enough that they can eventually stop getting kernel updates. Keystone2 and GZ/G1H are among those that still need kernel updates on 8GB machines for the moment, i.MX6 with 4GB is in a similar situation.
  * Compressed swap pages in the highmem area might still be used on later kernels for that hardware, instead of directly providing high memory pages to user space.
  * SoCs based on Armv7VE cores (Cortex-A7, -A15, and A17) in configurations with 2-4GB of RAM can move to the work-in-progress support for `CONFIG_VMSPLIT_4G_4G` that will allow accessing close to 4GB of physical memory while also providing 4GB of virtual memory for processes. This comes at a higher performance cost than highmem but, since it is based on features of the Armv7 MMU, will not be nearly as bad as the comparable feature that was used on x86 in the past.
  * The same trick as Armv7VE might also be used on Baikal T1 with its MIPS32r5 processor, but is harder to implement there. Older MIPS32 machines only support up to 512 MB of low memory due to hardware limitations.
  * Other 32-bit architectures can use `CONFIG_VMSPLIT_2G_OPT` to make up to 2GB of physical memory available without highmem support, at the cost of reduced user virtual memory. This would include Cortex-A9 as well as x86 and PowerPC machines, on which a `CONFIG_VMSPLIT_4G_4G` option is unlikely to get implemented because of hardware limitations or lack of interest.

If the plans for `CONFIG_VMSPLIT_4G_4G` work out, less than 1% of the machines should lose support for newer kernels after highmem gets removed, and that number goes down as time passes. Without `CONFIG_VMSPLIT_4G_4G` on the other hand, losing support for a portion of the 32-bit machines might not be acceptable at all, so highmem would have to remain an option for as long as users update their Armv7 kernels. Either way, this topic will remain interesting for years to come, as compromises continue to be made based on shifts in priority.

#### 64-Bit hardware

The continued move towards 64-bit hardware means that most users no longer have to worry about the problems of 32-bit systems such as the `time_t` overflow and limits of the virtual and physical address space, but this is not true for everyone. For example: 

  * There are a lot of legacy binaries that users may need to keep running, in particular on x86 machines, that have 30 years of backward compatibility including games ported from 32-bit Windows
  * Similarly, Android apps are often still being distributed as 32-bit Arm binaries for the widest hardware compatibility in some markets, despite Google's push for apps in its own Play store to be built for 64-bit or architecture-independent.
  * As 64-bit processors take over the low-cost market, they get combined with smaller memory configurations, with as little as a single 2Gb DDR3 system-in-package configuration for 256MB of physical memory. At this point, running 64-bit code has a significant overhead and there is a strong cost incentive to run 32-bit code instead of doubling the memory capacity.

The most common way to deal with this problem is to run a 64-bit kernel but boot into an unmodified file system containing a 32-bit user space. This works because the kernel has long supported both types of applications by implementing a second set of system call entry points known simply as the "compat" layer for each interface that is incompatible. There are a few exceptions in device drivers that do not support compat tasks, but those are mostly limited to third-party modules outside of the mainline kernel. 

The same compat kernel support allows mixing 32-bit and 64-bit container instances, as well as mixing both types of applications inside of a single "multilib" or "multiarch" root filesystem. Using compat mode avoids most of the problems with highmem, but still requires compiling all the 32-bit user application code with a time64-capable C library to be able to run beyond year 2038. 

On x86 and MIPS machines, another option is to run a 32-bit kernel directly on 64-bit hardware, which provides better compatibility with device drivers and lets users share both the kernel and root file system across 32-bit and 64-bit hardware, but brings back the virtual address limitations of the 32-bit kernel and will lack kernel features that are only available in 64-bit builds. In theory, this is also possible on many Arm processors, but there are many limitations: 

  * Armv8 processors lack certain instructions (`setend`, `cp15` barriers, `swp`/`swpb` atomics) that may be used in older 32-bit binaries. While the 64-bit kernel emulates these in compat mode, the 32-bit kernel emulation is less complete, so existing binaries that work on 64-bit Armv8 kernels as well as 32-bit Armv7 systems might not work on the same 32-bit kernel running on Armv8 hardware.
  * Most CPU implementations require some errata workarounds in the kernel to work correctly. The 32-bit Arm kernel has workarounds for all Armv7 cores, but not for Armv8 cores. This can result in undefined behavior and data corruption in the worst case.
  * Many security features require 64-bit kernels, either because of hardware limitations or missing implementation in 32-bit kernels. This includes [kernel page-table isolation](<https://lwn.net/Articles/741878/>) ("Meltdown" protection), [kernel address-space layout randomization](<https://lwn.net/Articles/569635/>), [PAN](<https://lwn.net/Articles/651614/>), [UAO](<https://developer.arm.com/docs/ddi0595/h/aarch64-system-registers/uao>), [BTI](<https://lwn.net/Articles/804982/#bti>), [PTR_AUTH](<https://lwn.net/Articles/718888/>), and hardware cryptography. Running a 32-bit kernel can therefore be less secure.
  * In general, this configuration is not well tested, as kernel developers rightfully point to using 64-bit kernels as a better alternative.
  * The latest generation of Armv8.2 cores have dropped hardware support for running 32-bit kernels completely. This includes the "little" Cortex-A55 as well as the all "big" cores starting with Cortex-A76. These can still run 32-bit user space code in compat mode. Some future cores will drop 32-bit user mode as well, but it seems likely that there will still be "little" cores for as long as there is demand for them. In fact it [appears](<https://lwn.net/Articles/838339/>) that there will be systems that can run 32-bit code only on some of the cores.

Despite these limitations, there have been large-scale deployments of 32-bit Arm kernels on 64-bit hardware, notably in the Raspbian distribution that manages to use the same code across Armv6 (Raspberry Pi 1 and Zero), Armv7-A (Raspberry Pi 2), and Armv8 (Raspberry Pi 3 and 4), as well as some low-end Android systems that appear to do this primarily to avoid the stricter Generic Kernel Image (GKI) rules that are required for shipping Android 10 with 64-bit Arm kernels. 

Memory consumption by the kernel itself may turn out to be a reason to finally add proper support for Armv8 processors to the 32-bit Arm kernel, to squeeze out the few last percent of usable memory in very small machines. As an estimate, there are multiple areas in which 64-bit kernels add an unavoidable overhead that can amount to dozens of megabytes:

  * The kernel image itself is roughly 50% larger compared to a Thumb2-enabled Armv7 kernel. On a typical configuration, it can grow from 10MB to 15MB, or more if the 64-bit configuration includes support for additional hardware as is the case with Android GKI.
  * As each 4KB page in the system requires a `struct page` entry that is typically 32 bytes larger on 64-bit kernels, one extra megabyte of memory is consumed for each 128MB of installed memory just to store the `mem_map` array.
  * Each thread needs its own kernel stack, which is 16KB instead of 8KB. This requires one extra megabyte for every 128 threads.
  * Page tables need one or two extra indirection levels to describe the larger virtual address space, adding at least 4KB per process.
  * Slab caches in the kernel tend to be larger as well; in particular the inode cache can appear to use significant memory resources. This is harder to quantify, as the inode cache is dynamically sized; it grows as long as memory is available but shrinks under memory pressure.

On processors that have no native 32-bit mode, it is also possible to define an "[ILP32](<https://docs.oracle.com/cd/E19620-01/805-3024/lp64-1/index.html>)" ELF format for 64-bit instructions. This is in fact available on both MIPS under the "n32" name and on x86 as "x32". Neither of the two is widely deployed in practice, and the patches to add the same for 64-bit Arm kernels were never merged, in order to avoid fragmenting the ecosystem with incompatible binary formats. 

It appears that the emerging RISC-V SoCs designed to run Linux offer only 64-bit mode. All the options available to Arm processors are theoretically available there as well, but in the absence of an established 32-bit RISC-V Linux ecosystem, it seems the architecture will end up largely skipping that step entirely and instead accept the higher memory consumption. There are a few 32-bit RISC-V cores that include an MMU, such as Rocket, Andes A25, VexRiscv, and Cloudbear BI-350, but so far, 32-bit RISC-V has only managed to make its way into microcontrollers, and while compat mode is theoretically possible with the architecture, neither the kernel nor actual CPU implementations support it. An extreme example here is [a recently announced low-cost SoC](<https://www.cnx-software.com/2020/11/09/xuantie-c906-based-allwinner-risc-v-processor-to-power-12-linux-sbcs>) using a 64-bit RISC-V CPU with as little as 64MB of RAM. 

For the moment, 64-bit Arm processors hold the advantage over RISC-V, as they have two mature software ecosystems they can run, using more space-efficient 32-bit user space for machines under 1GB. As memory requirements and capacities grow, more applications will run better when built for 64-bit and this difference becomes less important.

#### Conclusion

As a general rule, cutting-edge technology is short-lived as it quickly gets replaced with the next generation, while anything that performs its function more cheaply than the competition can remain popular for a long time, possibly decades. This effect can be seen with Motorola 68000 architecture, the Arm9 CPU core, and the Raspberry Pi Zero board, which are all outliving generations of superior replacements. This effect gives hope to those waiting for the end of highmem, as the only hardware with memory configurations that require it was cutting-edge five or more years ago and can be expected to be retired within the next five years. 

The MMU-less microcontrollers and the smallest memory configurations for embedded systems may die a different death in the same time frame, as these are often deployed in highly customized environments without a plan for kernel updates. Once the companies using them have stopped deploying systems on new kernels, it may finally end, the same way that once-popular architectures like Arm7TDMI or Blackfin disappeared in Linux. 

In between those two extremes are a large number of embedded SoCs that are still popular today, with Cortex-A7-based designs over time becoming the only viable option for new 32-bit SoCs. These are cheaper and better than anything below them, while anything more powerful is in turn being replaced by 64-bit products. Once a 64-bit core beats the Cortex-A7 on performance, power consumption, and cost, this last market will move on to that for new products and 32-bit Linux kernels will only be needed for supporting existing users. New board designs are likely to appear for five to ten years after the last Armv7 chip has been created, but after that it can take another decade before the remaining users finally stop upgrading their kernels or replace their hardware.
