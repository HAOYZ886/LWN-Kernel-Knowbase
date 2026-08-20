---
title: The  conclusion of the 2.6.36 merge window
url: https://lwn.net/Articles/400074/
date: "August 16, 2010"
category: "Releases-2.6.36"
author: "By Jonathan Corbet August 16, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 16, 2010

The 2.6.36 merge window closed with the release of 2.6.36-rc1 on August 15\. About 1000 changesets were merged into the mainline after [last week's update](<http://lwn.net/Articles/399052/>); this article will cover the significant additions since then, starting with the user-visible changes: 

  * The Squashfs filesystem has gained support for filesystems compressed with LZO. 

  * The Ceph filesystem now has advisory locking support. 

  * There is now support for erase and trim operations (including the "secure" variants) in the multi-media card (MMC) subsystem. The block layer has been extended with a new `REQ_SECURE` flag and a new `BLKSECDISCARD` `ioctl()` command to support this functionality. 

  * New drivers: 

    * **Block** : ARM PXA-based PATA controllers. 

    * **Systems and processors** : Income s.r.o. PXA270 single-board computers, Wiliboard WBD-111 boards, and Samsung S5PC210-based systems. 

    * **Miscellaneous** : Semtech SX150-series I2C GPIO expanders, Intersil ISL12022 RTC chips, Freescale IMX DryIce real time clocks, Dallas Semiconductor DS3232 real-time clock chips, SuperH Mobile HDMI controllers, iPAQ h1930/h1940/rx1950 battery controllers, Intel MID battery controllers, STMicroelectronics STMPE I/O expanders, TI TPS6586x power management chips, ST-Ericsson AB8500 power regulators, Maxim Semiconductor MAX8998 power management controllers, Intersil ISL6271A power regulators, Analog Devices AD5398/AD5821 regulators, Lenovo IdeaPad ACPI rfkill switches, Winbond/Nuvoton NUC900 I2C controllers, and SMSC EMC2103 temperature and fan sensors.  

Changes visible to kernel developers include: 

  * The `ioctl()` file operation has been removed, now that all in-tree users have been converted to the `unlocked_ioctl()` version which does not acquire the big kernel lock. Removal of the BKL has gotten yet another step closer. 

  * The nearly unused function `dma_is_consistent()`, meant to indicate whether cache-coherent DMA can be performed on a specific range of memory, has been removed. 

  * The kfifo API has been [reworked](<http://lwn.net/Articles/347619/>) for ease of use and performance. Some examples of how to use the API have been added under `samples/kfifo`. 

  * There is a new set of functions for avoiding races with sysfs access to module parameters: 

```
kparam_block_sysfs_read(name);
            kparam_unblock_sysfs_read(name);
            kparam_block_sysfs_write(name);
            kparam_unblock_sysfs_write(name);
```

Here, `name` is the name of the parameter as supplied to `module_param()` in the same source file. They are implemented with a mutex. 

  * Experimental support for multiplexed I2C busses has been added. 

All told, some 7,770 changes were incorporated during this merge window. There were not a whole lot of changes pushed back this time around. The biggest feature which was not merged, perhaps, was transparent hugepages, but that omission is most likely due to the lack of a proper pull request from the maintainer. 

Now the stabilization period begins. Linus has suggested that he plans to repeat his attempt to hold a hard line against any post-rc1 changes which are not clearly important fixes; we will see how that works out in practice.
