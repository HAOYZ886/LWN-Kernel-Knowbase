---
title: Safe PCI hot removal
url: https://lwn.net/Articles/88807/
date: "June 9, 2004"
category: Hotplug; PCI
author: 
---

The PCI hotplug mechanism promises improved server availability; when hotplug is used, PCI peripherals can be added to or removed from the system without taking the server down. As one developer [found out](<https://lwn.net/Articles/88808/>) recently, however, hotplug can also lead to the opposite result. Some devices have drivers which, if the device is removed before being closed, will crash the system. Surely, he asks, this is not the way things are supposed to be? 

The [answer](<https://lwn.net/Articles/88810/>) that came back indicated that, technically, this is a fine state of affairs. By the PCI hotplug specification, devices are supposed to be closed down before removal, and the operating system is not required to deal properly with the opposite sequence of events. This is, in other words, a "don't do that" situation. 

That said, it is generally possible for drivers to handle a too-hot unplugging of a device. A certain degree of care is required, however. Essentially, a driver for a hot-removable device must check for errors every time it attempts to communicate with that device. An error reading from or writing to a device register is usually the first indication that the device has left the building. When such errors happen, the driver must respond accordingly: error out any outstanding operations and mark the device as being unavailable. 

Over time, drivers with this kind of problem will get fixed. In the mean time, however, much driver code still shows signs of having been written when hardware additions and removals required a screwdriver and a power-down. When doing run-time surgery on an important system, it is still important to step carefully.
