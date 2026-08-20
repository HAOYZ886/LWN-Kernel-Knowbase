---
title: Tracking functional dependencies between devices
url: https://lwn.net/Articles/710494/
date: "January 3, 2017"
category: Power management
author: "By Jonathan Corbet January 3, 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 3, 2017

Computing systems have grown significantly in complexity since the Linux kernel was first written. In response, the kernel has developed new mechanisms for managing device complexity, including the driver model, dynamic number assignment, and more. These mechanisms have solved a number of problems but, while the problem of managing runtime dependencies between seemingly independent devices has been understood for some time, it didn't get a proper solution until the 4.10 merge window. 

Some device dependencies are inherent in the architecture of the system. For example, a USB peripheral will not be usable if the appropriate USB host adapter is unavailable, and that adapter is probably connected to some other system bus that must also be up and running. Dependencies based on the connection topology of the system are relatively easily represented in a tree structure; that is what the kernel's device model was created to do. Using this model, the kernel can, for example, suspend devices in the system in the correct order, keeping intermediate devices operational until all the devices that depend on them have been shut down. 

In a modern system, though, the dependency graph can be rather more complicated. A camera "device", for example, is likely to be a set of interconnected devices that look independent to the kernel. Actually operating the camera requires a sensor device, which is probably controlled via a connection to an I2C bus; it probably also depends on a couple of GPIO devices for its power and reset lines. The sensor is connected to a separate bridge device that acquires the image data; that bridge may need a DMA controller to move that data into memory. There may be other devices for various hardware-implemented image transformations (rotation or color-space conversion, for example) in the mix as well. 

The point is that each of these components looks like a separate device to the kernel. These devices are on separate controllers and, perhaps, on separate buses; they do not appear to be related from a look at the topology of the system. For the most part, a top-level driver marshals these devices together and makes them function together; the information it needs to do this task is, in current systems, often found in the device tree structure. But the kernel's driver core can break things if it shuts down one of the subdevices because it doesn't understand that other devices depend on that subdevice. 

Drivers have tended to work around this problem with one-off device-specific code. As one might expect, that leads to a fair amount of code duplication and a lot of inadequate solutions. It would be better to have a single solution in the driver core code that works for all devices. Moving toward that solution is the objective of the [functional dependencies](<https://static.lwn.net/kerneldoc/driver-api/device_link.html>) infrastructure merged for the 4.10 kernel. 

The interface to this mechanism is relatively simple, consisting of a single function to indicate that a dependency exists: 

```
struct device_link *device_link_add(struct device *consumer,
    				    	struct device *supplier,
    					u32 flags);
```

This call informs the driver core that the `consumer` device depends on the `supplier` device. So, for example, the system will not suspend `supplier` unless `consumer` is already suspended, and it will not probe or resume `consumer` until `supplier` is up and functional. Additionally, if the `supplier` device is unbound, the `consumer` device will, by virtue of no longer being able to function anyway, be unbound automatically. 

By default, device links are persistent and will remain in place for as long as the system is running. If, however, the `DL_FLAG_AUTOREMOVE` flag is provided when the link is created, that link will be automatically removed if the driver of the consumer device is unbound. These non-persistent links can be useful in situations where the hardware can be configured in multiple ways, creating varying dependencies over time. The `DL_FLAG_STATELESS` flag can be used to create a link that is used for suspend/resume ordering, but which is not otherwise managed by the driver core. 

If there is a need to explicitly remove a device link, that can be done with a call to `device_link_del()`: 

```
void device_link_del(struct device_link *link);
```

As of 4.10-rc2, there is only one user of this new infrastructure (the Exynos I/O memory-management unit code) in the mainline kernel. There will certainly be others that will show up in future development cycles, though. With luck, they will be accompanied by a reduction in driver-specific dependency-handling code and an improvement in kernel quality overall.
