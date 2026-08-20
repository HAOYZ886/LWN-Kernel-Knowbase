---
title: "Power management: looking for a direction"
url: https://lwn.net/Articles/98420/
date: "August 18, 2004"
category: Power management; Software suspend
author: 
---

Power management remains one of the unfinished jobs from the 2.5 development series. Many of the pieces are in place, including the whole device model infrastructure, but the kernel still lacks a comprehensive, working power management subsystem. There are signs that things are starting to happen, but it seems that the developers still lack a clear idea of how they want to go forward. 

Back on August 9, Patrick Mochel posted [a patch](<https://lwn.net/Articles/97142/>) aimed at improving the power management subsystem. It brought significant changes to the device model, including: 

  * Two power management methods were added to the class subsystem. Until this point, classes had not been part of the power management code at all; they are, instead, a way of exporting device information in a functional organization. The rationale behind putting power management functions in classes was that the higher-level code would better understand how to "quiesce" a device in preparation for a power state change. 

  * Three new power management methods were be added to the device model representation of a bus (`struct bus_type`). These were `pm_save()` (save state prior to a state transition), `pm_restore()` (restore state afterward), and `pm_power()` (perform an actual state change). These methods would replace the current `suspend()` and `resume()` bus methods, and the equivalent methods associated with `struct device_driver`. The idea is to move all power management tasks firmly into the bus-level code, and to let that code pass things on to low-level drivers as appropriate. 

  * Each device would get two new arrays. One of these (`pm_supports`) lists all of the power management states supported by the device, in that particular device's (usually bus-specific) terms. The second array (`pm_system`) is a simple mapping from the power states understood system-wide into the equivalent device states. These states are described by the new `pm_state` structure, and sysfs interfaces exist to query the supported states and to transition between them. 

The resulting discussion implied a lot of changes to this patch; among other things, the idea of using the class layer to quiesce devices was controversial. An updated version of the patch has not been posted, however. 

Pavel Machek, meanwhile, has been trying to address a much smaller piece of the problem: confusion over what the power management states really mean. The power management code itself uses a set of states roughly related to those defined in the ACPI specification, but other parts of the system (PCI drivers, for example) have a different set of states. The current power management methods take a `u32` state value, and it is far from clear what kind of state is intended. 

Pavel's [patch](<https://lwn.net/Articles/98430/>) tries to address this problem by creating a new `enum` type called `system_state`. The bus- and driver-level power management methods are modified to accept a parameter of this type, so that it is clear that (1) the power management core's state values are being used, and (2) the parameter describes the state to which the entire system is changing. It clears up a core ambiguity without otherwise changing how things work. 

Even this change is controversial, however. The largest concern is that, eventually, it is expected that the drivers will need more information than just the target system state. So, it is suggested, the type of the parameter should be a structure pointer rather than a simple scalar value. But nobody has really figured out what should go into the structure yet. 

Getting it right the first time matters in this case. It is generally accepted that fixing power management will require a driver API change, and that, potentially, all drivers in the kernel (and out of tree as well) will have to be changed at once. Developers are resigned to this change - but they would really rather only do it once. So, [says Patrick](<https://lwn.net/Articles/98436/>), it's better to wait: 

Why be hasty? We need to do it right and do it once. If that means a couple of more weeks and several more emails, than so be it. Otherwise, we'll be stuck with a sub-par solution for who knows how long. 

What this means is that the discussion is likely to continue for a while - and that an upgraded power management system will not be ready until 2.6.10, at best. Linux users, who have waited a long time for better power management, can probably manage to be patient for a little while yet.
