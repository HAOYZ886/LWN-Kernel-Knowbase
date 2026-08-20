---
title: "OLS: Three talks on power management"
url: https://lwn.net/Articles/240253/
date: "June 30, 2007"
category: ACPI; Power management
author: 
---

Len Brown can only be a glutton for punishment; he is, after all, the maintainer of the Linux ACPI subsystem. That is a difficult position to be in: ACPI involves getting into the BIOS layer, an area of system software which is not always known for careful, high-quality work. Supporting ACPI is a complex task which, among other things, requires the embedding of a specialized interpreter within the kernel, a hard sell at best. Even with [![\[Len Brown\]](https://static.lwn.net/images/conf/ols2007/LenBrown-sm.jpg)](<https://lwn.net/Articles/240335/>) that background in mind, one must wonder just how much masochism is required to lead one to deliver three separate talks at the 2007 Ottawa Linux Symposium. That is just what Len did, however; the end result was a good view into several aspects of the power management problem. 

### Getting more from tickless

The first talk (on the tickless kernel) was supposed to be given by Suresh Siddha, who was unable to attend the event. The [dynamic tick patches](<http://lwn.net/Articles/223185/>) have been covered here before. Suresh/Len's talk was not really about how these patches work, but, instead, about the work which remains to be done to take full advantage of the tickless design. It seems that the work which has been done so far is just the beginning. 

The problem is that, on a system used by Suresh and company, the average processor sleep time was still less than 1ms even after the dynamic tick code was enabled. Given that one of the driving reasons for dynamic tick was to let the processor sleep for long periods of time - thus saving power \- this is a disappointing result. It turns out that there is a lot which can be done to improve the situation, though. 

Step number one is to address a kernel-space problem: there are a lot of active kernel timers which tend to spread out over time. As a result, the kernel wakes up much more often than it would if the timers were sufficiently well coordinated to expire at the same time whenever possible. As it happens, many kernel timers do not need great precision; a timer which fires some number of milliseconds later than scheduled is not a problem. So, if the kernel could defer some timers to fire at the same time as others, it can reduce the number of wakeups. The [deferrable timers](<http://lwn.net/Articles/228143/>) patch does exactly that; the `round_jiffies()` function added in 2.6.19 can also help the kernel line up events. Adding this code brought the average sleep time up to 20ms, with the system handling 90 interrupts per second. 

Next is the problem of hardware timers. On the i386 architecture, the preferred timer is the local APIC (LAPIC) timer, which is built into the processor and very fast to program. Unfortunately, putting the processor into a deep sleep also puts the LAPIC timer to sleep, a situation Len compared to unplugging one's alarm clock before going to bed. In either case, oversleeping can be the unwanted result. The programmable interval timer (PIT) remains awake and is easily used, but it has a maximum event time of 27ms. If one wants the processor to sleep for longer than that, another solution must be found. That solution is the high-precision event timer (HPET), which has a maximum interval of at least three seconds. Getting access to the HPET can be hard, though; good BIOS support is spotty at best and the HPET is often disabled. If it can be forced on, however, the system can go to an average sleep period of about 56ms, handling 32 interrupts per second. 

Even better is to get the HPET out of the "legacy mode" currently used by Linux. This mode is simple to use, but it requires the rebroadcasting of timer interrupts on multiprocessor systems. But the HPET can work with per-CPU channels, eliminating this problem. The result: average sleep time grows to 74ms. 

At this point, the problem moves to user space. Since the release of [powertop](<http://www.linuxpowertop.org/>), there has been a lot of progress in this area; user-space applications which cause frequent wakeups unnecessarily stand out immediately and can be fixed. But, as Len noted, "user space still sucks." 

### ACPI myths

One gets the sense that Len is a little tired of people complaining about ACPI in Linux. His response was a talk on "ten ACPI myths" - though the list had grown to twelve by then. 

**#1: There is no benefit to enabling ACPI**. Len's answer to this had two parts, the first of which being that, increasingly, there is no alternative. The older APM interface is deprecated, and, in particular, Microsoft's Vista has removed APM support altogether. So, soon, there will be no hardware support for APM at all; it is a dead standard. The MPS standard (used for discovering processors) is also old and dying. Like it or not, ACPI is needed to be able to make use of one's hardware. 

On the positive side, using ACPI gives better access to hardware features like software-enabled power, sleep, and lid buttons. Smart battery status information becomes available, as well as the potential for reduced power consumption and better battery life. True hotplug and (especially) docking support also become possible with ACPI. 

**#2: Suspend-to-disk problems are ACPI's fault**. In fact, ACPI is a very small part of the suspend-to-disk process - everything else is in other parts of the kernel code. If you have suspend-to-disk problems, suggests Len, "complain to Pavel [Machek], not me." 

**#3: If the extra buttons don't work, it's ACPI's fault**. The issue here is that support for "hotkeys" is not actually a part of the ACPI specification. All of those extra buttons found on laptops are vendor-specific added features. The reverse-engineered drivers currently found in the kernel are a "heroic effort," but they should not be necessary. Vendors should be supplying drivers for their hardware. 

**#4: Boot problems with ACPI enabled are ACPI's fault**. Len allows that this one might just be true some of the time. But disabling ACPI at boot-time also disables other hardware features - the IO-APIC in particular. So any problems associated with those other parts of the system will be masked by turning off ACPI. It looks like ACPI was the actual problem, but the truth is more complicated. 

**#5: ACPI issues are due to sub-standard platform BIOS**. It turns out that there are three general sources of ACPI incompatibilities. Just one of them is the BIOS violating the ACPI specification; incompatibilities which don't break Windows will often slip through the testing process. The [firmware developer kit](<http://www.linuxfirmwarekit.org/>) produced by Intel can help in this regard. Another source of problems is differing interpretations of the specification, which is a long and complex document. The Linux ACPI developers have been working to help clarify the specification when this sort of problem arises. Finally, there can also simply be bugs in the Linux-specific code. 

**#6: The Linux community cannot help to improve the ACPI specification**. In fact, the ACPI team has been submitting improvements, mostly in the form of "specification clarifications." Many of those have been incorporated and shipped with specification updates. 

**#7: The ACPI code changes a lot but is not getting better**. Intel has put together a test suite with over 2000 tests; ACPI changes must now pass that suite before being merged. The number of new bug reports has been dropping - though, perhaps, more slowly than one might like. 

**#8: ACPI is slow and bad for high-performance CPU governors**. The ACPI interpreter is not used in performance-critical paths, and, thus, cannot be slowing things down. ACPI's role is in the setup and configuration process. 

**#9: The speedstep-centrino governor is faster than acpi-cpufreq**. The acpi-cpufreq governor has seen considerable improvements, and is now able to access MSRs in a fast and (more importantly) supportable way. So its performance is where it should be, and the speedstep-centrino governor is scheduled for removal. 

**#10: More CPU idle power states is better**. This may be true for any given processor, but you cannot compare processors on the basis of how many idle states they provide. All that really matters is how much power you save when you use those states. 

**#11: Processor throttling will save energy**. The problem here is a confusion of "power" and "energy." A throttled processor may draw less power, but it has to run longer to accomplish the same work. So throttling the processor (while maintaining the same voltage) may have the effect of increasing energy use rather than reducing it. The better approach is almost always to run at the fastest clock frequency afforded by the current voltage level and get the work done quickly; Len characterized this as the "race to idle." 

There are second-order effects to consider; in particular, batteries will last longer if they are discharged over longer periods of time. A throttled processor may also run cooler, allowing fans to be turned off. Throttling may be necessary for temperature regulation. But, from an energy-savings perspective, these are truly second-order effects. 

**#12: I can't contribute to improving ACPI in Linux**. Like any other project, Linux ACPI would love to have more developers. And, failing that, one can always test kernels and report bugs. There is, in reality, plenty of opportunity for improving the ACPI code. 

### Cool-hand Linux

Len's final talk moved away from power consumption toward its effects - the generation of heat, in particular. The creation of excess heat is not a welcome behavior in any device, but it becomes especially undesirable in handheld devices. Devices which make the user's hand sweat are less fun to use; those which get too hot to hold comfortably can be entirely unusable. So temperature management is important. But the nature of these devices can make thermal regulation tricky: there's no room for fans in a Linux-powered cellular phone, and the dissipation of heat can be hard in general. 

The ACPI 3.0 specification includes a complicated thermal model. The device is divided up into zones, and each component has its thermal contribution to each zone characterized. Implementing this specification is a complex and difficult task - enough so that the Linux ACPI developers have no intention of doing it. They will, instead, focus on something simpler. 

That something is the ACPI 2.0 thermal model. It includes thermal zones, each of which comes with temperature sensors and a set of trip points. The "critical shutdown" trip point is set somewhere just short of where the device begins to melt; should things get that warm, the device just needs to turn itself off as quickly as possible. Various other trip points will be encountered first; they should bring about increasingly strong measures for controlling temperature. These can include turning on fans (if they exist), throttling devices, or suspending the system to disk. ACPI 2.0 includes an embedded controller which monitors the system's temperature sensors and sends events to the CPU when something interesting happens. 

The in-progress thermal regulation code just uses the existing critical shutdown mechanism built into ACPI. There is also support for some of the passive trip points which bring about CPU throttling. For the non-processor thermal zones, though, the best thing to do is to let user space figure out how to respond, so that's what the ACPI code will do. There will be a netlink interface through which temperature events can be sent, and a set of sysfs directories for reading sensor values. The sysfs tree will also include control files which can be used by a user-space daemon to throttle specific devices in response to temperature events. 

In the end, the kernel is really just a conduit, conducting events and control settings between the components of the device and user space. There were some questions on whether there will be a standardized set of sysfs knobs for every device; the answer appears to be "no." Each device is different, with its own control parameters; it is hard to create any sort of standard which can describe them all. Beyond that, the target environment is embedded devices, each of which is unique. It is expected that each device will have its own special-purpose management daemon designed especially for it, so there is no real benefit in trying to make things generic. 

The impression one gets from all these talks is that quite a bit is happening in the power management area - a part of Linux which, for some time, has been seen as falling short of what it really needs to be. The increasing use of Linux in embedded systems can only help in this regard; there are a number of vendors who have a strong interest in improved support for intelligent use of power. Given time and continued work, power management may soon be one of those past problems which is no longer an issue.
