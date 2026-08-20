---
title: Closing out the 2.6.28 merge window
url: https://lwn.net/Articles/304592/
date: "October 27, 2008"
category: "Releases-2.6.28"
author: "By Jonathan Corbet October 27, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 27, 2008

About 1000 changesets were merged after [the previous summary](<http://lwn.net/Articles/303609/>) was posted here. Much of those came from architecture-specific trees. Other changes merged this time around include: 

  * There are new drivers for Mellanox ConnectX 10GbE network adapters, PowerPC PPC40x and PPC44x GPIO controllers, Panasonic "Let's Note" laptop special keys, Sharp SL-6000 backlight and LCD devices, Dialog Semiconductor DA9030/DA9034 backlight devices, Tabletkiosk Sahara Touch-iT backlight devices, and Toshiba TX4939 SoC ATA controllers. 

  * One more not-ready-for-prime-time driver was merged via the staging tree; this one supports Redrapids Pocket Change cardbus devices. The staging tree also brought an extensive set of fixes to the drivers added earlier in the merge window. 

  * The kernel has gained support for [ultra-wideband](<http://en.wikipedia.org/wiki/Ultra-WideBand>) protocol stacks. UWB can be used for normal networking, but the immediate application is [wireless USB](<http://lwn.net/Articles/241980/>), which will be supported in 2.6.28. 

  * The ACPI docking station code has gained support for bay and battery hotplug events. 

  * The IA64 architecture now supports Xen. Also added to IA64 is support for DMA remapping devices (IOMMUs). 

  * Support for [kdump](<http://lwn.net/Articles/108595/>) has been added to the PowerPC architecture. 

  * The 9P (Plan9) filesystem now has RDMA support. 

Changes visible to kernel developers include: 

  * There is a new `core_param()` macro: 

```
core_param(name, var, type, perm);
```

Its purpose is to define "core" parameters and let them be represented in `/sys/module/kernel/parameters`. 

  * It is now possible to create a workqueue running at realtime priority with: 

```
struct workqueue_struct *create_rt_workqueue(const char *name);
```

  * The block driver API has changed considerably, with the `inode` and `file` parameters being removed from most block device operations. The new API looks like this: 

```
struct block_device_operations {
        	int (*open) (struct block_device *bdev, fmode_t mode);
        	int (*release) (struct gendisk *gd, fmode_t mode);
        	int (*locked_ioctl) (struct block_device *bdev, fmode_t mode, 
        	    		     unsigned cmd, unsigned long arg);
        	int (*ioctl) (struct block_device *bdev, fmode_t mode, 
        	    	      unsigned cmd, unsigned long arg);
        	int (*compat_ioctl) (struct block_device *bdev, fmode_t mode, 
        	    		     unsigned cmd, unsigned long arg);
        	int (*direct_access) (struct block_device *bdev, sector_t sector,
        			      void **kaddr, unsigned long *pfn);
        	int (*media_changed) (struct gendisk *gd);
        	int (*revalidate_disk) (struct gendisk *gd);
        	int (*getgeo)(struct block_device *bdev, struct hd_geometry *geo);
        	struct module *owner;
            };
```

The new prototypes do away with the `file` and `inode` structure pointers which were passed in previous kernels. Note that the `ioctl()` method is now called without the big kernel lock; code needing BKL protection must explicitly define a `locked_ioctl()` function instead. 

  * The [range timer API](<http://lwn.net/Articles/296578/>) has been merged; callers can now specify a time period in which they would like the timeout to be delivered. The kernel can then take advantage of the range to coalesce wakeups and keep the processor idle for longer periods. 

This time around, linux-next maintainer Stephen Rothwell has put together [a list of linux-next patches](<http://lwn.net/Articles/304690/>) which did not get into 2.6.28. Perhaps the biggest omission was the [credentials work](<http://lwn.net/Articles/251469/>), which seemed poised to go in this time around. Other changes which failed to get merged include the [message catalog](<http://lwn.net/Articles/292650/>) code (which looks like it will need a change of approach) and [TOMOYO Linux](<http://lwn.net/Articles/277833/>) (which seems to be caught up in the same old "new security module with pathname-based rules" swamp). 

Now the stabilization period starts. Linus, perhaps, was trying to set the tone for this development cycle when he released a much smaller and earlier [2.6.28-rc2](<http://lwn.net/Articles/304748/>) than would have normally been expected. By way of comparison: 2.6.25-rc2 had 359 patches applied since 2.6.25-rc1. For 2.6.26-rc2, 446 changesets were merged, and, for 2.6.27-rc2, the count was 780. For 2.6.28-rc2, instead, a total of 22 changes went in. Says Linus: 

And hey, maybe we can even _continue_ the nice model of "just small fixes after -rc1". I know, it sounds insane, but it's a real pleasure to do an -rc2 with just a handful of fixes for real problems that real people see. What a concept! 

Should this pattern hold, it may well be that 2.6.28 will stabilize more quickly and successfully than its predecessors. It will, in any case, be interesting to watch.
