---
title: Dynamic USB device IDs
url: https://lwn.net/Articles/160944/
date: "November 21, 2005"
category: Device drivers; USB
author: 
---

The market for USB devices is certainly dynamic; new gadgets are released at a high rate. Unfortunately, Linux kernels and their associated drivers are not always updated quite as quickly. The result can be that the kernel fails to recognize and drive a new gadget, even though existing drivers may be entirely capable of doing the job. The driver simply does not know that the device is one it can handle, so the kernel does not bind the two together. 

Greg Kroah-Hartman has posted [a simple patch](<http://lwn.net/Articles/160785/>) which should help fix this situation. With the patch in place, each USB driver gets a new sysfs attribute (`new_id`). If a system administrator writes two values (the vendor and product ID numbers reported by the device) to that attribute, those numbers form a new device ID associated with the driver. Immediately after the write, the driver will recognize the device, and everybody will be happy. No changes to the drivers themselves are necessary. Of course, one could create confusion by associating a device with an inappropriate driver, but a bit of attention should suffice to avoid that problem. 

This patch came out a bit late for 2.6.15, so it is more likely to show up in 2.6.16 or thereafter.
