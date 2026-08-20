---
title: "Power-management knobs"
url: https://lwn.net/Articles/662701/
date: "November 2, 2015"
category: Power management
author: "By Jonathan Corbet November 2, 2015 2015 Kernel Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 2, 2015

* * *

[2015 Kernel Summit](<https://lwn.net/Articles/KernelSummit2015/>)

Rafael Wysocki started off his 2015 Kernel Summit session by noting that every generation of hardware promises to be more power-efficient than its predecessor. But that efficiency is not always experienced by users. In an ideal world, systems should run in the most power-efficient mode whenever possible and only employ the less-efficient modes when performance requirements demand it. Real-world systems, though, tend not to run as efficiently as they can. Rafael came with a proposal that, he thought, might improve the situation, but it's not clear that the idea will be implemented. 

Any hardware (be it a CPU or a peripheral device) with multiple operating states has to be able to detect when it is safe to change between modes. Almost all hardware has some power-related modes that it is able to employ, but, increasingly, power-efficient operation is not something that a device can choose independently from the rest of the system. A modern system-on-chip (SoC) includes a lot of devices in a single [![](https://static.lwn.net/images/conf/2015/klf-ks/RafaelWysocki-sm.jpg)](<https://lwn.net/Articles/662702/>) package; these devices share the same power-distribution mechanism. Often, many or all devices must go into a low-power state before the shared power hardware can change state, so a single misconfigured device can keep much of the system in a high-power state. And, as it happens, devices are often configured to not use their lower-power states. 

This configuration is done for a number of reasons, but it mostly comes down to safety. Some hardware can misbehave when running in a low-power state, and the consequences of that misbehavior can be severe. Sometimes the problem is in the driver, which does not handle low-power states well. Power-management support tends to be seen, rightly or wrongly, as being experimental, so developers often disable it. And sometimes there are legitimate performance concerns that argue against the use of low-power states. 

Power-management configuration can be tricky; there are a lot of knobs in the system that affect it in one way or another. There is usually at least one knob for every driver, often more; few users even know how to find all of these knobs, much less configure them properly. Rafael would like to make life easier for users wanting to tune their systems for power use; to that end, he has proposed the addition of a high-level switch that would enable power management globally. This switch would have to be a command-line parameter, since it needs to be set at boot time. 

The switch would have the two obvious settings: "performance" and "power." There could also be a setting to enable power-management settings that are still seen as experimental; braver users could turn that on as well. 

There were concerns expressed about the boot-time switch, since it cannot be changed in the `powertop` utility. A few developers suggested that, rather than adding a global knob, developers should make it easier for users to discover and set power-management options. Standardizing the operation of device-specific power-management knobs would help, as would better documentation of the various knobs found in sysfs. Knobs that are known to be experimental or dangerous should be explicitly marked as such; that might give users more confidence when it comes to tweaking the safer knobs. 

Darren Hart suggested that a global power-management policy could be implemented in user space if all of the relevant settings were standardized, discoverable, and well documented. Rafael pointed out that the [runtime power management](<https://lwn.net/Articles/347573/>) knobs are already standardized, but they still tend to remain set in the "disabled" state. Still, the sense of the group seemed to be that better discoverability and documentation were preferable to a global knob implemented in the kernel. So future work is likely to be focused in that area.
