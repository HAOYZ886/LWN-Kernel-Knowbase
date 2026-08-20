---
title: Idling ACPI idle
url: https://lwn.net/Articles/390427/
date: "June 1, 2010"
category: ACPI; Power management
author: "By Jonathan Corbet June 1, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 1, 2010

Len Brown has spent some years working to ensure that Linux has top-quality ACPI support. So it is interesting to see him working to short out some of that ACPI code, but that is what has happened with his new cpuidle driver for recent Intel processors. With this experimental feature - merged for 2.6.35 - Linux no longer depends on ACPI to handle idle state transitions on Nehalem and Atom processors. 

The ACPI BIOS is the standard way of getting at processor idle states in the x86 world. So why would Linux want to move away from ACPI for its cpuidle driver? Len [explains](<https://lwn.net/Articles/390428/>): 

ACPI has a few fundamental flaws here. One is that it reports exit latency instead of break-even power duration. The other is that it requires a BIOS writer to get the tables right. Both of these are fatal flaws. 

The motivating factor appears to be a BIOS bug shipping on Dell systems for some months now which disables a number of idle states. As a result, Len's test system takes 100W of power when using the ACPI idle code; when idle states are handled directly, power use drops to 85W. That seems like a savings worth having. The fact that Linux now uses significantly less power than certain other operating systems - which are dependent on ACPI still - is just icing on the cake. 

In general, it makes sense to use hardware features directly in preference to BIOS solutions when we have the knowledge necessary to do so. There can be real advantages in eliminating the firmware middleman in such situations. It's nice to see a chip vendor - which certainly has the requisite knowledge - supporting the use of its hardware in this way.
