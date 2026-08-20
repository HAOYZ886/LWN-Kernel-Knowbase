---
title: The conclusion of the 3.4 merge window
url: https://lwn.net/Articles/489665/
date: "April 3, 2012"
category: "Releases-3.4"
author: "By Jonathan Corbet April 3, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
April 3, 2012

Linus [announced](<https://lwn.net/Articles/490034/>) the 3.4-rc1 release and the closing of the merge window on March 31\. At the outset, he had said that this merge window could run a little longer than usual; in fact, at 13 days, it was slightly shorter. One should not conclude that there was not much to pull, though; some 9,248 non-merge changesets went into the mainline before 3.4-rc1, and a couple of significant features have sneaked their way in afterward as well. 

User-visible features merged since [last week's summary](<https://lwn.net/Articles/488131/>) include: 

  * The device mapper "thin provisioning" target now supports discard requests, a feature which should help it to use the underlying storage more efficiently. 

  * The [dm-verity](<https://lwn.net/Articles/459420/>) device mapper target has been merged. This target manages a read-only device where all blocks are checked against a cryptographic hash maintained elsewhere; it thus provides a certain degree of tampering detection. Details can be found in [Documentation/device-mapper/verity.txt](<https://lwn.net/Articles/489671/>)

  * Support for the [x32 ABI](<https://lwn.net/Articles/456731/>) has been merged into the kernel. Getting support into the compiler and the C library is an ongoing project, and the creation of distributions using this ABI will take even longer, but the foundation, at least, is now in place. 

  * The "high-speed synchronous serial interface" (HSI) framework has been merged. HSI is an interface that is mainly used to connect processors with cellular modem engines; it will be used for handset support in future kernel releases. 

  * New drivers include: 

    * **Processors and platforms** : Samsung EXYNOS5 SoCs, and NVIDIA Tegra3 SoCs. 

    * **Flash** : SMI-attached SPEAR MTD NOR controllers, DiskOnChip G4 NAND flash devices, and Universal Flash Storage host controllers (details in [Documentation/scsi/ufs.txt](<https://lwn.net/Articles/490046/>)). 

    * **Miscellaneous** : Apple "gmux" display multiplexers, Intel Sodaville GPIO controllers, TI TPS65217 and TPS65090 power management controllers, Ricoh RC5T583 power management system devices, Freescale i.MX on-chip ANATOP controllers, Summit Microelectronics SMB347 battery chargers, and ST Ericsson AB8500 battery management controllers.  

Changes visible to kernel developers include: 

  * The "common clock framework" unifies the handling of subsystem clocks, especially on the ARM architecture (though it is not limited to ARM). See [Documentation/clk.txt](<https://lwn.net/Articles/489668/>) for more information. 

  * The DMA buffer sharing API has been extended to allow CPU access to the buffers; see the updated [Documentation/dma-buf-sharing.txt](<https://lwn.net/Articles/489703/>) file for details. 

  * The direct rendering subsystem has gained initial support for the DMA buffer sharing mechanism. No drivers use it yet, but having this support in the mainline will ease the development of driver support for future kernels. 

  * The massive `<asm/system.h>` include file has been split into several smaller files and removed; in-tree users have been fixed. 

  * The new `/proc/dma-mappings` file on the ARM architecture displays the currently-active coherent DMA mappings. Since such mappings tend to be in short supply on ARM, this can be a useful debugging tool. 

  * The ARM architecture has gained jump label ("static branch") support. 

  * The just-in-time compiler for BPF packet filters has been ported to the ARM architecture. 

There are a couple of other features that Linus may still be considering merging as of this writing, though the chances of them getting in would appear to be diminishing. One is the [DMA mapping rework](<https://lwn.net/Articles/467509/>); Linus has been asking for potential users of this change to speak up, but few have done so. In other words, if there are developers out there who would like to see the improved DMA subsystem in the 3.4 release, you are running out of time to make that desire known. The other is [POHMELFS](<https://lwn.net/Articles/480095/>), which has had some review snags and which also seems to lack a vocal community clamoring for its inclusion. 

Beyond those possibilities, though, the time for new features to go into the 3.4 development cycle has now passed. The stabilization process has begun, with a probable final release in late May or early June.
