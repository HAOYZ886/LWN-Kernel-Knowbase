---
title: A single power preference knob
url: https://lwn.net/Articles/393288/
date: "June 23, 2010"
category: Power management
author: "By Jonathan Corbet June 23, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
June 23, 2010

Power management under Linux is getting more complex as the kernel's capabilities grow. It is now possible to control power use through scheduling policies, idle state management, device states, and so on. Unfortunately, some power management choices have performance consequences; depending on the use to which the system is being put, those consequences may not be welcome. So there must be a way for system administrators to control how power management decisions are made. 

Currently, that control is exercised through a number of individual system parameters. One controls whether the scheduler tries to coalesce processes onto a subset of the system's CPUs in the hope of letting others sleep. Another knob tells the idle governor which sleep states it is able to use. Yet another controls CPU frequency and voltage response. Simply knowing about all of the available parameters is hard; keeping them all tuned properly can be harder yet. 

Len Brown has proposed the addition of [an overall control parameter](<https://lwn.net/Articles/393289/>) for power management, to be found in `/sys/power/policy_preference`. This knob would have five settings, ranging from "maximum performance at all times" to "save as much power as possible without actually turning the system off." With a control like this, system administrators could control system power policy without having to learn about all of the individual parameters involved; policy choices would also be applied to any new power-management parameters added in the future. 

The idea was not universally loved, though. Some commenters asked for more than five settings, but Len argued that anybody needing more complex configurations should just continue to use the individual parameters. Others fear that the single policy might be interpreted differently by different drivers, leading to inconsistent results; they would rather see the continued use of individual parameters which exactly describe the desired behavior. The real discussion, though, cannot happen until some actual code has been posted, if and when that happens.
