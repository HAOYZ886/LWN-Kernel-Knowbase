---
title: The last things through the 2.6.26 merge window
url: https://lwn.net/Articles/281029/
date: "May 5, 2008"
category: "Releases-2.6.26"
author: "By Jonathan Corbet May 5, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 5, 2008

About 500 changesets were merged after the publication of the [first](<http://lwn.net/Articles/278965/>) and [second](<http://lwn.net/Articles/280034/>) 2.6.26 merge window summaries. The merge window is now closed; here is the final set of changes which got in: 

  * New drivers for Solarflare Communications Solarstorm SFC4000 controller-based Ethernet controllers, Hauppauge HVR-1600 TV tuner cards, ISP 1760 USB host controllers, Cypress c67x00 OTG controllers, and Intel PXA 27x USB controllers. 

  * 8Kb stacks are, once again, the default for the x86 architecture. ""Out-of-memory situations are less problematic than silent and hard to debug stack corruption."" 

  * The klist type now has the usual-form macros for declaration and initialization: `DEFINE_KLIST()` and `KLIST_INIT()`. Two new functions (`klist_add_after()` and `klist_add_before()`) can be used to add entries to a klist in a specific position. 

  * As had been planned, `struct class_device` has been removed from the driver core, along with all of the associated infrastructure. Classes are now implemented with an ordinary `struct device`. 

  * `kmap_atomic_to_page()` is no longer exported to modules. 

  * There are some new generic functions for performing 64-bit integer division in the kernel: 

```
u64 div_u64(u64 dividend, u32 divisor);
            u64 div_u64_rem(u64 dividend, u32 divisor, u32 *remainder);
            s64 div_s64(s64 dividend, s32 divisor)
            s64 div_s64_rem(s64 dividend, s32 divisor, s32 *remainder);
```

Unlike `do_div()`, these functions are explicit about whether signed or unsigned math is being done. The x86-specific `div_long_long_rem()` has been removed in favor of these new functions. 

  * There is a new string function: 

```
bool sysfs_streq(const char *s1, const char *s2);
```

It compares the two strings while ignoring an optional trailing newline. 

  * The prototype for i2c `probe()` methods has changed: 

```
int (*probe)(struct i2c_client *client, 
                          const struct i2c_device_id *id);
```

The new `id` argument supports i2c device name aliasing. 

  * There is a new configuration (`MODULE_FORCE_LOAD`) which controls whether the loading of modules can be forced if the kernel thinks something is not right; it defaults to "no."
