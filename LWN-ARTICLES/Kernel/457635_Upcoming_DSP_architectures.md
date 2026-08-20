---
title: Upcoming DSP architectures
url: https://lwn.net/Articles/457635/
date: "September 7, 2011"
category: Architectures
author: "September 7, 2011 This article was contributed by Arnd Bergmann"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

September 7, 2011

This article was contributed by Arnd Bergmann

The Linux-3.2 merge window may be the first time that two new CPU architectures get merged at the same time: the [c6x architecture](<http://linux-c6x.org>) from Texas Instruments and the [Hexagon architecture](<http://lwn.net/Articles/455342/>) from Qualcomm. Following the recently merged [OpenRisc](<http://opencores.org/openrisc,linux>) platform, the two submissions look very solid and should see no major obstacles getting merged into Linux after the usual review comments have been resolved, but there is still [some debate](<http://www.spinics.net/lists/linux-arch/msg14597.html>) over how to best add glibc support for the architectures. 

Interestingly, there is a lot that these two architectures have in common, far beyond [coincidentally implementing the same bugs](<http://www.spinics.net/lists/linux-arch/msg14558.html>). Both are not regular CPU cores designed to run an operating systems but instead are essentially [digital signal processors](<http://en.wikipedia.org/wiki/Digital_signal_processor>), similar to the Blackfin architecture that was merged into linux-2.6.22 a few years ago. Further, both Hexagon and c6x are already widely available in systems running Linux on the ARM core of a TI OMAP or Qualcomm MSM system-on-a-chip, where they are used for offloading CPU intensive work such as that required for video codecs. 

It will be interesting to see how Linux can coexist in the long run when the same SoC can run Linux on either of the two CPU architectures. The ARM architecture is currently transitioning to probing based on the [dts device tree format](<https://lwn.net/Articles/447918/>) and all new architectures merged into Linux will have to use that format as well when they have devices that cannot be automatically detected. If the device tree vision comes true, a single board will actually be able to use the same device tree binary on either one, independent of which CPU actually runs the kernel. 

Another intriguing scenario is running Linux on both architectures (ARM and DSP) simultaneously, using shared memory to communicate between them. Ohad Ben-Cohen has recently posted [a framework based on virtio](<https://lwn.net/Articles/448562/>) to allow just that on the TI platform. While virtio was intended to be used for communication between a virtual machine and the host operating system, it turns out to be flexible enough to allow the same drivers for communication between operating system instances on the same hardware. 

Looking closer at the actual DSP architectures, there are some major differences between Hexagon and C6x. The former is quite capable, with support for symmetric multiprocessing, a memory management unit and even a hypervisor. It can be seen as a competitor to established CPU architectures like ARM, x86 or powerpc, at least in the embedded space. In contrast, C6x is a rather minimalistic architecture dating back to the TMS320 introduced in 1983. So far, its kernel supports neither SMP nor an MMU, which means it is restricted to running [μClibc](<http://uclibc.org/>) instead of glibc, and it has a very limited set of applications that can be supported as long as it is missing the MMU. 

Beyond Linux 3.2, there are still more architectures that have been around for a long time and could get merged if the respective maintainers were interested. FPGA-based [Nios2](<http://gitorious.org/linux-nios2>) is apparently close to getting submitted, while the similar [lm32 architecture](<http://www.theobroma-systems.com/uclinux-lm32/>) saw a lot of activity in 2010 but does not seem to be actively worked on now. [Synopsys ARC](<http://openlinux.amlogic.com/wiki/index.php/Arc>) and [Imagination META](<http://www.imgtec.com/meta/meta.asp>) are both claimed to have Linux and Android support, but there is no indication that the authors are actively working on upstream submission or even on making the patches for current kernels easily available. Finally, Donald Knuth's [MMIX architecture](<http://www007.upp.so-net.ne.jp/eiji-y/vmmmix/vmmmix.html>) has seen some occasional work in the past but now appears to be stalled, the latest kernel source version being 2.6.18.
