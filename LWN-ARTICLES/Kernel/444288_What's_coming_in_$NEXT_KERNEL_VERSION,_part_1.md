---
title: "What's coming in $NEXT_KERNEL_VERSION, part 1"
url: https://lwn.net/Articles/444288/
date: "May 25, 2011"
category: "Releases-3.0"
author: "By Jonathan Corbet May 25, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 25, 2011

As of this writing, some 5400 non-merge changesets have been pulled into the mainline kernel for the next release. The initial indications are that this development cycle will not have a huge number of exciting new features, but there are still some interesting additions. Among the user-visible changes are the following: 

  * There are two new POSIX clock types: `CLOCK_REALTIME_ALARM` and `CLOCK_BOOTTIME_ALARM`; they can be used to set timers that will wake the system from a suspended state. See [this article](<https://lwn.net/Articles/429925/>) for more information on these new clocks. 

  * The [Quick Fair Queue packet scheduler](<http://info.iet.unipi.it/~luigi/qfq/>) has been added to the network stack. 

  * The [just-in-time compiler for BPF packet filters](<https://lwn.net/Articles/437981/>) has been merged; only x86-64 is supported for now. 

  * There is a new networking system call: 

```
int sendmmsg(int fd, struct mmsghdr *mmsg, unsigned int vlen,
                         unsigned int flags);
```

It is the counterpart to `recvmmsg()`, allowing a process to send multiple messages with a single system call. 

  * The [ICMP sockets](<https://lwn.net/Articles/420799/>) feature has been merged; its main purpose is to allow unprivileged programs to send echo-request datagrams. 

  * Two new sysctl knobs allow the capabilities given to user-mode helpers invoked by the kernel to be restricted; see [the commit](<http://git.kernel.org/linus/17f60a7da150fdd0cfb9756f86a262daa72c835f>) for details. 

  * The tmpfs filesystem has gained support for extended attributes. 

  * The Xen block backend driver (allowing guests to export block devices to other guests) has been merged. 

  * New hardware support includes: 

    * **Systems and processors** : Netlogic XLR/XLS MIPS CPUs, Lantiq MIPS-based SOCs, PowerPC A2 and "wire speed processor" CPUs, and Armadeus APF9328 development boards. 

    * **Audio/video** : Philips TEA5757 radio tuners, Digigram Lola boards, Apple iSight microphones, Maxim max98095 codecs, Wolfson Micro WM8915 codecs, Asahi Kasei AK4641 codecs, HP iPAQ hx4700 audio interfaces, NXP TDA18212 silicon tuners, Micron MT9V032 sensors, Sony CXD2820R DVB-T/T2/C demodulators, RedRat3 IR transceivers, Samsung S5P and EXYNOS4 MIPI CSI receivers, and Micronas DRXD tuners. 

    * **Input** : PenMount dual touch panels, Maxim max11801 touchscreen controllers, Analog Devices ADP5589 I2C QWERTY keypad and I/O expanders, and Freescale MPR121 Touchkey controllers. 

    * **Network** : Marvell "WiFi-Ex" wireless adapters (SD8787 initially) and Marvell 8787 Bluetooth interfaces. 

    * **USB** : Renesas USBHS controllers, Samsung S5P EHCI controllers, Freescale USB OTG transceivers, and Samsung S3C24XX USB high-speed controllers. 

    * **Miscellaneous** : CARMA DATA-FPGA programmers, Broadcom's "advanced microcontroller bus architecture," Freescale SEC4/CAAM security engines, Samsung S5PV210 crypto accelerators, Maxim MAX16065, MAX16066, MAX16067, MAX16068, MAX16070, and MAX16071 system managers, Maxim MAX6642 temperature sensors, TI UCD90XXX system health controllers, TI UCD9200 system controllers, Analog Devices ADM1275 hot-swap controllers, Analog Devices AD5504, AD5501, AD5760, and AD5780 DACs, Analog Devices AD7780 and AD7781 analog to digital convertors, Analog Devices ADXRS450 Digital Output Gyroscopes, Xilinx PS UARTs, TAOS TSL2580, TSL2581, and TSL2583 light-to-digital converters, Intel "management engine" interfaces, nVidia Tegra embedded controllers, and IEEE 1588 (precision time protocol) clocks. 

Also added to the staging tree is the user-space support code for the [USB/IP subsystem](<http://usbip.sourceforge.net/>) which allows a system to "export" its USB devices over the net. 

Changes visible to kernel developers include: 

  * Prefetching is no longer used in linked list and hlist traversal; this may be the beginning of a much more extensive program to remove explicit prefetch operations. See [this article](<https://lwn.net/Articles/444336/>) for more information on the prefetch removal. 

  * There is a new `strtobool()` function for turning user-supplied strings into boolean values: 

```
int strtobool(const char *s, bool *res);
```

Anything starting with one of `[yY1]` is considered to be true, while strings starting with one of `[nN0]` are false; anything else gets an -EINVAL error. 

  * There is a whole series of new functions for converting user-space strings to kernel-space integer values; all follow this pattern: 

```
int kstrtol_from_user(const char __user *s, size_t count, 
        			  unsigned int base, long *res);
```

These functions take care of safely copying the string from user space and performing the integer conversion. 

  * The kernel has a new generic binary search function: 

```
void *bsearch(const void *key, const void *base, size_t num, size_t size,
        	          int (*cmp)(const void *key, const void *elt));
```

This function will search for `key` in an array starting at `base` containing `num` elements of the given `size`. 

  * The use of threads for the handling of interrupts on specific lines can be controlled with `irq_set_thread()` and `irq_set_nothread()`. 

  * The [`static_branch()` interface for the jump label mechanism](<https://lwn.net/Articles/436041/>) has been merged. 

  * The function tracer can now support multiple users with each tracing a different set of functions. 

  * The alarm timer mechanism - which can set timers that fire even if the system is suspended - has been merged. 

  * An object passed to `kfree_rcu()` will be handed to `kfree()` after the next read-copy-update grace period. There are a lot of RCU callbacks which only call `kfree()`; it should be able to replace those with `kfree_rcu()` calls. 

  * The -Os (optimize for size) option is no longer the default for kernel compiles; the associated costs in code quality were deemed to be too high. Linus said: ""I still happen to believe that I$ miss costs are a major thing, but sadly, -Os doesn't seem to be the solution. With or without it, gcc will miss some obvious code size improvements, and with it enabled gcc will sometimes make choices that aren't good even with high I$ miss ratios."" 

  * The first rounds of ARM architecture cleanup patches have gone in. A number of duplicated functionalities have been consolidated, and support for a number of (probably) never-used platform and board configurations have been removed. 

  * The `W=` parameter to kernel builds now takes values from 1 to 3\. At the first level, only warnings deemed to have a high chance of being relevant; a full kernel build generates "only" 4800 of them. At `W=3`, developers get a full 86,000 warnings to look at. Note that if you want _all_ of the warnings, you need to say `W=123`. 

The merge window for this development cycle is likely to end on May 29, just before Linus boards a plane for Japan. At that time, presumably, we will learn what the next release will be called; Linus has made it clear that he thinks the 2.6.x numbers are getting too high and that he thinks it's time for a change. Tune in next week for the conclusion of this merge window and the end of kernel version number suspense.
