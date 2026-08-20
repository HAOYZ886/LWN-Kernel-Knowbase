---
title: The end of the 4.0 merge window
url: https://lwn.net/Articles/634471/
date: "February 24, 2015"
category: "Releases-4.0"
author: "By Jonathan Corbet February 24, 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 24, 2015

By the time Linus released [4.0-rc1](<https://lwn.net/Articles/634346/>) on February 22, 8,950 non-merge changesets had been pulled into the mainline repository for this development cycle. The changes pulled are the usual mix for the end of the merge window, with fixes starting to dominate over new features. Still, a few new things were to be found in the 1,100 changes pulled since [last week's summary](<https://lwn.net/Articles/633096/>), including: 

  * The overlayfs union filesystem can now support multiple read-only layers. 

  * The [virtio subsystem](<https://lwn.net/Articles/239238/>) has been updated for compliance with the [recently adopted](<https://lwn.net/Articles/580186/>) virtio 1.0 standard. 

  * The Btrfs filesystem has received a set of out-of-space-handling fixes resulting from its use at Facebook. The pull request suggests there will be more of these coming in the future. 

  * The dm-crypt device mapper target has seen a number of scalability improvements that [improve its performance](<https://www.redhat.com/archives/dm-devel/2015-February/msg00106.html>) on larger systems. 

  * New hardware support includes: 

    * **Systems and processors** : Intel Quark X1000 SoC boards and MIPS processors running MIPS32 Release 6\. 

    * **Clock** : TI CDCE706 clock synthesizers and Qualcomm IPQ806x and APQ8064/MSM8960 LPASS clock controllers. 

    * **Miscellaneous** : Hisilicon NAND flash controllers, Renesas R-Car Gen2 DMA controllers, IMG multi-threaded DMA controllers, Allwinner SoC pulse-width modulator (PWM) controllers, Imagination Technologies PWM controllers, Intel Baytrail I2C semaphores, and Broadcom iProc I2C controllers. 

    * **Power management** : Richtek RT5033 power management ICs, Dialog Semiconductor DA9150 charger fuel-gauge chips, and Qualcomm resource power managers. 

    * **Watchdog** : Imagination Technologies PDC watchdog timers and Mediatek SoC integrated watchdogs.  

The indications at the beginning of the merge window were that this would be a relatively small development cycle. In fact, as can be seen in the table below, one has to go back to 3.6 (released in September 2012) to find a merge window with fewer patches: 

> Patches pulled during  
> the merge window  
> ---  
> Release| Patches  
> 4.0| 8,950  
> 3.19| 11,408  
> 3.18| 9,711  
> 3.17| 10,872  
> 3.16| 11,364  
> 3.15| 12,034  
> 3.14| 10,622  
> 3.13| 10,518  
> 3.12| 9,479  
> 3.11| 9,494  
> 3.10| 11,963  
> 3.9| 10,265  
> 3.8| 10,901  
> 3.7| 10,409  
> 3.6| 8,587  
> 3.5| 9,534  
> 3.4| 9,248  
> 3.3| 8,899  
> 3.2| 10,214  
> 3.1| 7,202  
> 3.0| 7,333  
  
From the table, one can see that there is a natural ebb and flow to the kernel development process; sometimes there is simply more going on than others. The overall trend remains in the upward direction, though, with the number of changes going into the kernel growing over the long (or even medium) term. 

As was expected, Linus has bumped the major version number of this release to "4". There is little significance to this change beyond the fact that the minor numbers were getting large. 

This development cycle has now moved into the stabilization phase where the remaining bugs are (hopefully) found and fixed. The last three development cycles have been exactly nine weeks long; if that pattern holds this time around as well, the 4.0 kernel will be released on April 12.
