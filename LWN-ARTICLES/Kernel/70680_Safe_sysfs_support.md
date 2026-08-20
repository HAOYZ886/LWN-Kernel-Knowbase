---
title: Safe sysfs support
url: https://lwn.net/Articles/70680/
date: "February 11, 2004"
category: Race conditions; Sysfs; class simple
author: 
---

It has long been intended that the sysfs virtual filesystem would contain information about all of the hardware (and more) installed on a given system. Implementation of this intention has lagged in places, however, and there are still parts of the system which lack sysfs support. One of those areas is the frame buffer device code. In an attempt to fill in that gap, James Simmons recently posted [a patch](<https://lwn.net/Articles/70681/>) adding sysfs support for frame buffer devices; this patch was merged into 2.6.3-rc1. 

There is only one problem with this patch: it can oops the kernel when frame buffer driver modules are unloaded. The problem is the same one which has afflicted other subsystem sysfs implementations: lifecycle rules. Once a data structure has been exposed via sysfs, user space can hold references to that structure indefinitely. Open sysfs files can persist long after the underlying device has been removed from the system, and long after the relevant module has been unloaded. If the behavior of sysfs-exposed data structures has not been carefully laid out, the kernel can be left holding references to structures or code which no longer exist. 

This sort of problem hit the networking subsystem hard. Once `net_device` structures were exposed via sysfs, it was no longer possible to allow individual network drivers to control what the lifecycle of those structures is. As a result, it is now necessary to allocate all `net_device` structures dynamically, and to let the networking subsystem decide when and how to free those structures. The networking code is also very careful not to access any module code after a `net_device` has been shut down. The end result is that `net_device` structures can persist in the system long after the module which created them has been removed. It all works, but the cost was a lengthy cleanup operation which has only now reached something close to completion. 

The frame buffer patches attempted to do things right from the beginning by making the `fb_info` structure into a dynamic object. A support function exists to allocate the structure, and it is automatically freed when the last reference is removed. The only problem is that the frame buffer drivers do not use this interface; they allocate and destroy `fb_info` structures on their own. As a result, in the 2.6.3-rc1 (and -rc2) kernel, `fb_info` structures can be freed twice (or staticly-allocated structures can be freed once). That sort of error tends to create displays on the frame buffer that the user does not want to see. 

Fixing this problem requires updating every frame buffer driver to use dynamically-allocated `fb_info` structures. James has stated his intent to make this change. In the mean time, the "stable" kernel release candidate has a known problem which will require a wide-ranging set of changes to fix. Al Viro, a master of this sort of transition, has [grumbled](<https://lwn.net/Articles/70686/>) that these changes should have been done in the opposite order, so as to avoid breaking things. Others have complained that this sort of change is too big for a stable kernel series and should have waited for 2.7. 

Yet another approach, however, would be to use the "class_simple" interface, which was merged in 2.6.2-rc1. This interface makes it easy to retrofit a `/sys/class` interface into existing drivers without having to deal with some of the more complex lifecycle issues. The interface is straightforward; one starts by creating a class: 

```
struct class_simple *class_simple_create(struct module *owner,
                                                 char *name);
```

The `owner` argument should almost always be passed as `THIS_MODULE`; the `name` will show up under `/sys/class`. The resulting class can be removed at some later time with: 

```
void class_simple_destroy(struct class_simple *class);
```

Entries for individual devices can be added with: 

```
struct class_device *class_simple_device_add(struct class_simple *class,
                                                     dev_t dev,
    						 struct device *device,
    						 const char *fmt, ...);
```

Here, `class` is the class which was created above, `dev` is the device number for the device, `device` is a `struct device` structure for this device (it can be `NULL`), and the rest is a `printk()`-style format string to create the name for the entry. The result (on success) is a sysfs directory with exactly one attribute: a file called `dev` which contains the device number. That is adequate for a tool like `udev` to create corresponding device nodes. 

The entry can be removed, of course: 

```
void class_simple_remove(dev_t dev);
```

The whole thing works without maintaining references into the calling driver, so most of the lifetime rule issues are avoided. More recent changes to the `class_simple` interface include (in 2.6.3-rc) hotplug support.
