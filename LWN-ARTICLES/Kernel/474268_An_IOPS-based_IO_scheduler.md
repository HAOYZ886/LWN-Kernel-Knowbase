---
title: "An IOPS-based I/O scheduler"
url: https://lwn.net/Articles/474268/
date: "January 4, 2012"
category: "Block layer-IO scheduling; Elevator; IO scheduler"
author: "By Jonathan Corbet January 4, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 4, 2012

I/O schedulers are charged with ordering block I/O operations in a way that maximizes throughput to the device and, perhaps, implementing the system's policy with regard to how the available bandwidth should be divided. The schedulers currently in use in Linux were designed with rotating storage in mind, with the result that they are concerned with avoiding disk seeks and tracking the number of bytes transferred. With solid-state devices, though, I/O locality is (nearly) irrelevant and the number of I/O operations performed is considered to be a better measurement of the amount of device capacity used. The kernel's CFQ scheduler has been evolving to deal better with solid-state devices, but everybody agrees there is more to be done. 

Shaohua Li has taken a new approach with the posting of [a new I/O scheduler](<https://lwn.net/Articles/474164/>) that is optimized for solid-state devices. The patch set factors out and generalizes the CFQ code that tracks device usage, but then uses that code to implement a different scheduling algorithm. Avoiding seeks is no longer a concern; neither is the number of bytes transferred. Instead, this scheduler simply tracks the number of I/O operations submitted by each user, trying to equalize the number from each. 

The result should be a simpler scheduler that is better suited to solid-state devices. At this point, though, it is hard to say for sure. One of the key rules of kernel patch submission is that performance-oriented changes should be accompanied by benchmark results showing that they achieve the intended goal. This patch had no such results, so nobody knows if it is worth their while to look at the code further or not. Presumably the next submission will provide that information, at which point the real discussion of the new scheduler's merits can begin.
