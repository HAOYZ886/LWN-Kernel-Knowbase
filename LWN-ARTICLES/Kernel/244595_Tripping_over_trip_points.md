---
title: Tripping over trip points
url: https://lwn.net/Articles/244595/
date: "August 7, 2007"
category: ACPI; Power management; Thermal management
author: "By Jonathan Corbet August 7, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 7, 2007

Contemporary processors have an interesting problem: if they operate at their full rated capacity for extended periods of time, they run a real risk of heating to the point that they let the blue smoke out and never run again. To avoid this kind of problem, processors (and other components) are instrumented with temperature sensors. The BIOS programs the sensors with specific "trip points" - temperatures where things will happen to keep the system from overheating. At a given trip point, the system might turn on the fan, throttle the processor, or, if disaster is imminent, shut the system down hard. 

The Linux ACPI subsystem provides the ability to query these trip points; the relevant virtual files can be found under `/proc/acpi/thermal_zone`. Your editor's laptop, for example, reveals that it is set to throttle the processor at 86°C and to pull the plug at 91°. Traditionally, the ACPI code has also allowed a suitably privileged user to change those trip points by writing new values to the `/proc` files. That capability no longer exists, though; it was removed in the 2.6.22 kernel. 

Users are now [starting to complain](<https://lwn.net/Articles/244601/>) about that change. They feel that the BIOS-set trip points on some systems are positioned incorrectly, resulting in systems that run more slowly than they think they should, fans which come on at the wrong time, and so on. Naturally, they feel that the removal of the trip-point override feature has reduced the functionality of their systems. 

ACPI maintainer Len Brown [responds](<https://lwn.net/Articles/244604/>) that the override feature is a bad idea for a number of reasons. At the top of the list is the fact that the system cannot actually change the hardware trip points. All it can do is disable them. Then the processor must take over by polling the temperature sensors itself and responding when its software trip points are reached. Should that polling and response fail to happen for any reason, there is a real possibility that the hardware could be damaged. Meltdowns could also easily occur if the trip points are set incorrectly, leading to "Linux destroyed my laptop" postings echoed across the net. 

On top of that, the BIOS can change the trip points at any time for reasons of its own. Many of the use cases for trip-point overrides (controlling when fans go on and off, for example) are better done by having a user-space daemon control fan operation directly. And the truth of the matter is that overriding trip points is usually (Len would say always) an inappropriate response to problems which are better solved somewhere else. When the issue was discussed in May, he [summarized](<http://lkml.org/lkml/2007/5/20/248>) it this way: 

The fact that the trip-points are writable has obscured, rather than clarified, the actual causes of the failures. No less than 4 people in that bug report declared that cleaning the dust out of their fan fixed the root cause. A bunch more said that the issues went away when they stopped using ubuntu's user-space power save daemon. 

There are a couple more with broken active fan control -- which also gets obscured rather than clarified by over-riding trip points. 

The remaining problems, says Len, are most likely not present when Windows is running on the affected hardware. And, he says, Windows is highly unlikely to be overriding the trip points. The conclusion is that Linux is doing something wrong in its thermal management on those systems. He would much rather find and fix the real problem than hide it through use of trip-point overrides. 

In the end, according to Len, there has never yet been a bug report which suggests that Linux should be messing with trip points in this way. This is a clear challenge for anybody who misses the trip-point override feature: send in a suitably documented report showing the problem that this feature solved. If the override feature truly turns out to be necessary, it may just come back - but it may just happen that a fix for the actual problem goes in instead.
