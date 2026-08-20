---
title: "Driver core functions: GPL only"
url: https://lwn.net/Articles/104393/
date: "September 29, 2004"
category: "Device model; Modules-Licensing; Sysfs"
author: 
---

Patrick Mochel may have been expecting to start a flame war with [this patch](<https://lwn.net/Articles/104392/>), which changes most of the driver core functions to be exported only to GPL-licensed modules. The affected functions include the bus-level code, classes (but not `class_simple`), `device_register()` and friends, the platform and system bus functions, low-level sysfs functions, and the kobject primitives. In fact, the flame war failed to materialize; nobody seems to be upset by these changes. Whether Patrick is pleased or disappointed by the silence is for him to say. 

The affected functions are a fundamental component of the Linux driver model; they are used by every device driver and filesystem, and by many other parts of the kernel as well. Even so, few, if any, proprietary modules will be affected by this change. The interfaces used by most modules are built on top of - and hide - the driver core. Thus, it is a rare driver which calls `device_register()`; instead, something like `usb_register_dev()` is used. Those upper-layer functions remain exported to all modules. 

So why make the change? Patrick's reasoning is that he wants all users of the low-level functions to be part of the mainline kernel tree. 

In short, being able to audit all of the users of these functions is necessary to their continued evolution (whatever that may entail). It would make the most sense if all users were part of the kernel, and it makes little sense to support their use by any unknown or binary modules. 

As the kernel tree becomes more dynamic internally, it will be increasingly hard for external modules - free or not - to keep up with the changes. It would not be surprising to see ever more "encouragement" to merge external modules into the mainline. Code which remains outside will require a higher level of maintenance, or it is likely to break frequently.
