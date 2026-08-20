---
title: "1½ Topics: realtime throttling and user-space adaptive spinning"
url: https://lwn.net/Articles/931789/
date: "May 13, 2023"
category: "Scheduler-Realtime; Spinlocks-User-space"
author: "By Jonathan Corbet May 13, 2023 OSSNA"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 13, 2023

* * *

[OSSNA](<https://lwn.net/Archives/ConferenceByYear/#2023-Open_Source_Summit_North_America>)

The Linux CPU scheduler will let realtime tasks hog the CPU to the exclusion of everything else — except when it doesn't. At the 2023 [Open Source Summit North America](<https://events.linuxfoundation.org/open-source-summit-north-america/>), Joel Fernandes covered the problems with the kernel's realtime throttling mechanism and a couple of potential solutions. As a bonus, since the room was unscheduled for the following slot, attendees were treated to a spontaneous session on adaptive spinning in user space run by André Almeida.
