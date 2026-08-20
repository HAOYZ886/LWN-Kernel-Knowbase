---
title: Some power management changes for 2.6.15
url: https://lwn.net/Articles/157949/
date: "November 1, 2005"
category: Power management
author: 
---

The 2.6.14 kernel has brought with it a few changes to the power management API. The first of these has to do with the `suspend()` and `resume()` methods found in `struct device_driver`. These methods would be called three times for each suspend and resume operation, in order to maintain compatibility with an older version of the API. The new versions are called once, and have different prototypes: 

```
int (*suspend) (struct device *dev, pm_message_t state);
        int (*resume) (struct device *dev);
```

This change required updates to a fair number of drivers, so the patch is relatively large. 

The other change is for devices which can supply "wakeup events" to the kernel. These devices include network adapters with "wake-on-LAN" capability, keyboards, and simple power switches. The power management core has been reworked to enable these devices to perform their wakeup functions while providing overall control to the system administrator. 

The `dev_pm_info` structure (found inside `struct device`) has gotten two new, single-bit fields. Drivers for devices which can create wakeup events should set the `can_wakeup` field to one. The actual issuance of such events, however, should be controlled by the `may_wakeup` field. If that field is zero, the power management core has decreed that wakeups should not be issued. A `device_may_wakeup()` helper function has been added to make testing the `may_wakeup` bit easy. 

The patch adds a new `wakeup` field in sysfs. When read, it will return `enabled` or `disabled` (or an empty string if the device is not capable of generating wakeup events at all). The system administrator can also write a new value to allow (or disallow) the generation of wakeup events from the device. 

The driver core code has been merged, along with support for wakeups from USB devices. As of this writing, however, the PCI wakeup code has some outstanding issues with G5 systems which has prevented it from going into the mainline.
