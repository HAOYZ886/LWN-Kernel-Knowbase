---
title: Controlling device power management
url: https://lwn.net/Articles/585439/
date: "February 12, 2014"
category: "Power management-Device power management"
author: "By Jonathan Corbet February 12, 2014"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 12, 2014

The kernel's power management code works to keep the hardware in the most power-efficient state that is consistent with the current demands on the system. Sometimes, though, overly aggressive power management can interfere with the proper functioning of the system; putting the CPU into a sleep state might wreck ongoing DMA operations, for example. To avoid situations like that, the [pm_qos](<https://lwn.net/Articles/386139/>) (power management quality of service) mechanism was added to the kernel; using pm_qos, device drivers can describe their requirements to the power-management subsystem. More recently, we have seen a bit of a change in focus in pm_qos, though, as it evolves to handle power management within peripheral devices as well. 

A partial step in that direction was taken in the 3.2 development cycle, when per-device constraints were added. Like the original pm_qos subsystem, this mechanism is a way for devices to specify their own quality-of-service needs; it allows a driver to specify a maximum value for how long a powered-down device can wait to get power back when it needs to do something. This value (called `DEV_PM_QOS_LATENCY` in current kernels) is meant to be used with the [power domains](<https://lwn.net/Articles/449585/>) feature to determine whether (and how deeply) a particular domain on a system-on-chip could be powered down. 

The quest for lower power consumption continues, though, and, as a result, we are seeing more devices that perform their own internal power management based on the access patterns they observe. Memory controllers might put some banks into lower power states if they are not seeing much use, for example; this technology seems to work well enough to take much of the wind out of the sails of the various [memory power management](<https://lwn.net/Articles/568891/>) patch sets out there. Disk drives can spin themselves down, camera sensors can turn themselves off, and so on. Peripherals do not have as good an idea of future access patterns as the host computer should, but, it turns out, they can often do a good job of guessing based on the recent past. 

That said, there will certainly be times when a device will decide to take a nap at an inopportune moment. To help avoid this kind of embarrassing situation, many devices that have internal power management provide a way for the host system to communicate its latency needs to the device. If such a device has been informed by the CPU that it should respond with a latency no greater than, say, 10ms, it will not go into any sleep states that would take longer to come back out of. 

Current kernels have no formalized way to control the latency requirements communicated to devices, though. That situation could change as early as the 3.15 development cycle, though, if Rafael Wysocki's [latency tolerance device pm_qos type patch set](<https://lwn.net/Articles/584541/>) finds its way into the mainline. This work uses much of the existing pm_qos framework, but to a different end: rather than allowing drivers to communicate their requirements to the power management core, this mechanism carries latency requirements back to drivers. 

The first step is to rename `DEV_PM_QOS_LATENCY`, which, it could be argued, has an ambiguous name in the new way of doing things. The new name (`DEV_PM_QOS_RESUME_LATENCY`) may not be that much clearer to developers reading the code from the outside, but it does make room for the new `DEV_PM_QOS_LATENCY_TOLERANCE` value. As noted above, this pm_qos type differs from the others in that it communicates a tolerance _to_ a device; it also differs in that it is exposed to user space. Any device that supports this feature will have a new attribute (`pm_qos_latency_tolerance_us`) in its sysfs `power` directory. A specific latency value (in µs) can be written to this attribute to indicate that the device must be able to respond in the given period of time. There are two special values as well: "`auto`", which puts the device into its fully automatic power-management mode, and "`any`", which does not set any specific constraints, but which tells the hardware not to adjust its latency tolerance values in response to other power-management events (transitions to and from a suspended state, for example). 

Device power management information is stored in `struct dev_pm_info` which, in turn, is found in `struct device`. Devices supporting `DEV_PM_QOS_LATENCY_TOLERANCE` need to provide a new function in that structure: 

```
void (*set_latency_tolerance)(struct device *dev, s32 tolerance);
```

Whenever the latency tolerance value changes, `set_latency_tolerance()` will be called with the new value. The special `tolerance` value `PM_QOS_LATENCY_ANY` corresponds to the "`any`" value described above. Otherwise, a negative `tolerance` value indicates that the device should be put into the fully automatic mode. 

In many cases, driver authors will not need to concern themselves with providing this callback, though. Instead, it will be handled at the bus level, perhaps in combination with the firmware. The initial implementation posted by Rafael takes advantage of the "latency tolerance reporting" registers provided via ACPI by some Intel devices; for such devices, the power management implementation exists in the ACPI code and need not be duplicated elsewhere. 

The final step is to actually make use of this feature when hardware that supports it is available. Such use seems most likely to show up in mobile systems and other dedicated settings where the software can easily be taught to tweak the latency parameters when the need arises. Writing applications that can tune those parameters on a general-purpose system seems like a harder task. But, even there, when the hardware wants to do the wrong thing, there will be a mechanism to set it straight.
