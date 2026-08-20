---
title: Reworking disk events handling
url: https://lwn.net/Articles/423619/
date: "January 19, 2011"
category: "Block layer-Block drivers; Device drivers-Block drivers"
author: "By Jonathan Corbet January 19, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 19, 2011

A look inside any contemporary desktop-oriented system is likely to reveal a process which is steadfastly polling removable drives on the off chance that somebody might have removed or inserted a disk. Indeed, as your editor can attest, it can be hard to turn that polling off; there's little room in the world for strange people who have their own ideas of what they want to happen when they put a disk into a drive. Be that as it may, if the system is going to poll drives, it would be nice to do so in the best way possible. That is not currently the case, but, thanks to a patch by Tejun Heo, drive polling should be better in 2.6.38. 

There are a few problems with how polling is done on Linux; these were nicely outlined by Tejun in [the patch changelog](<http://git.kernel.org/linus/77ea887e433ad8389d416826936c110fa7910f80>). Polling on Linux requires opening the device; this is a somewhat heavyweight operation which does not naturally line up with other operations which might wake the processor. Opening the device in this way might interfere with other users; optical disk burning, in particular, is susceptible to this kind of problem. And polling the disk in this way generates a different set of commands than Windows uses; as Linux driver developers have discovered many times, behavior that differs from Windows is not well tested by vendors and tends to have unpleasant bugs. All that notwithstanding, user-space polling works well enough most of the time, but it would still be nice to make it better. 

Tejun's patch works by moving the polling into the kernel. That makes the polling more efficient by removing the need to open the device and by allowing the kernel to delay polling wakeups until something else is going on as well. There is a new function added to `struct block_device_operations` which should be implemented by drivers: 

```
unsigned int (*check_events) (struct gendisk *disk, unsigned int clearing);
```

This function should check the device for new events and return a mask of any which were found. Two events are currently defined: `DISK_EVENT_MEDIA_CHANGE` and `DISK_EVENT_EJECT_REQUEST`, the latter of which is new. The `clearing` parameter is a mask of events which should be cleared until they happen again. 

The old `media_changed()` block device operation still exists, but its use has been deprecated; drivers should be updated to use `check_events()` instead. Drivers should also, before adding a block device, initialize two new `struct gendisk` fields: 

```
unsigned int events;
        unsigned int async_events;
```

A mask of all events which can be reported by the device should be stored in `events`, while `async_events` should list the events which can be reported without needing to poll the device. 

A new sysctl knob (`block.events_dfl_poll_msecs`) tells the kernel how often it should (by default) poll block devices. A value of zero (the default, currently) disables polling entirely. Polling intervals for specific devices can be set in their sysfs directories. If a device says that it can report all events asynchronously, and no polling interval has been explicitly set for it, that device will not be polled at all. 

Since user space is no longer polling the device with this scheme, it needs a new way to find out when a disk event has happened. These events are now signaled via a uevent, meaning they can be handled via udev or some other utility which is watching those events. Note that any driver which handles asynchronous event reporting must call `kobject_uevent_env()` itself to send the event to user space. No driver in 2.6.38-rc1 does that; the first developer to add such a call may want to add a helper function to the core block code for that purpose. 

Since polling is disabled by default, the kernel will behave the way it always has and existing user space applications will work. Once the user space environments have been changed to take advantage of this feature, they can turn on kernel polling and stop opening the devices themselves. That should lead to better power consumption and more reliable operation, which can only be a good thing.
