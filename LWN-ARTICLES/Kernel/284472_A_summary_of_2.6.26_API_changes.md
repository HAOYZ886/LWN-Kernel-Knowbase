---
title: A summary of 2.6.26 API changes
url: https://lwn.net/Articles/284472/
date: "June 11, 2008"
category: "Releases-2.6.26"
author: "By Jonathan Corbet June 11, 2008"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
June 11, 2008

The 2.6.26 development cycle has stabilized to the point that it's possible to look at the internal API changes which have resulted. They include: 

  * At long last, support for the KGDB interactive debugger has been added to the x86 architecture. There is a DocBook document in the Documentation directory which provides an overview on how to use this new facility. Some useful features (e.g. KGDB over Ethernet) are not yet supported, but this is a good start. 

  * Page attribute table (PAT) support is also (again, at long last) available for the x86 architecture. PATs allow for fine-grained control of memory caching behavior with more flexibility than the older MTRR feature. See [Documentation/x86/pat.txt](<https://lwn.net/Articles/278994/>) for more information. 

  * `ioremap()` on the x86 architecture will now always return an uncached mapping. Previously, it had taken a more relaxed approach, leaving the caching as the BIOS had set it up. The practical result was to _almost_ always create uncached mappings, but with occasional exceptions. Drivers which depend on a cached mapping will now break; they will need to use `ioremap_cache()` instead. See [this article](<http://lwn.net/Articles/282250/>) for more information on this change and caching in general. 

  * The [generic semaphores patch](<http://lwn.net/Articles/273731/>) has been merged. The semaphore code also has new `down_killable()` and `down_timeout()` functions. 

  * The final users of `struct class_device` have been converted to use `struct device` instead. The `class_device` structure, along with its associated infrastructure, has been removed. 

  * The `nopage()` virtual memory area operation has been removed; all in-tree code is now using `fault()` instead. 

  * The [object debugging infrastructure](<http://lwn.net/Articles/271614/>) has been merged. 

  * Two new functions (`inode_getsecid()` and `ipc_getsecid()`), added to support security modules and the audit code, provide general access to security IDs associated with inodes and IPC objects. A number of superblock-related LSM callbacks now take a `struct path` pointer instead of `struct nameidata`. There is also a new set of hooks providing generic audit support in the security module framework. 

  * The now-unused ieee80211 software MAC layer has been removed; all of the drivers which needed it have been converted to mac80211. Also removed are the sk98lin network driver (in favor of skge) and bcm43xx (replaced by b43 and b43legacy). 

  * The `ata_port_operations` structure used by libata drivers now supports a simple sort of operation inheritance, making it easier to write drivers which are "almost like" existing code, but with small differences. 

  * A new function (`ns_to_ktime()`) converts a time value in nanoseconds to `ktime_t`. 

  * Greg Kroah-Hartman is no longer the PCI subsystem maintainer, having passed that responsibility on to Jesse Barnes. 

  * The seq_file code now accepts a return value of `SEQ_SKIP` from the `show()` callback; that value causes any accumulated output from that call to be discarded. 

  * The Video4Linux2 API now defines a set of controls for camera devices; they allow user space to work with parameters like exposure type, tilt and pan, focus, and more. 

  * On the x86 architecture, there is a new configuration parameter which allows gcc to make its own decisions about the inlining of functions, even when functions are declared `inline`. In some cases, this option can reduce the size of the kernel's text segment by over 2%. 

  * The legacy IDE layer has gone through a lot of internal changes which will break any remaining out-of-tree IDE drivers. 

  * A condition which triggers a warning from `WARN_ON` will now also taint the kernel. 

  * The `get_info()` interface for `/proc` files has been removed. There is also a new function for creating `/proc` files: 

```
struct proc_dir_entry *proc_create_data(const char *name, mode_t mode,
        					    struct proc_dir_entry *parent,
        					    const struct file_operations *proc_fops,
        					    void *data);
```

This version adds the `data` pointer, ensuring that it will be set in the resulting `proc_dir_entry` structure before user space can try to access it. 

  * The klist type now has the usual-form macros for declaration and initialization: `DEFINE_KLIST()` and `KLIST_INIT()`. Two new functions (`klist_add_after()` and `klist_add_before()`) can be used to add entries to a klist in a specific position. 

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

One change which did _not_ happen in the end was the change to 4K kernel stacks by default on the x86 architecture. This is still a desired long-term goal, but it is hard to say when the developers might have enough confidence to make this change.
