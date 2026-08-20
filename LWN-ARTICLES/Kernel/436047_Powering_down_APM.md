---
title: Powering down APM
url: https://lwn.net/Articles/436047/
date: "March 30, 2011"
category: Power management
author: "By Jonathan Corbet March 30, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 30, 2011

The APM power management interface has never been much loved - even ACPI was seen as a better alternative. There has been little or no hardware made which depends on APM for some years; Windows evidently stopped supporting it in 2006. Linux does still support APM, though, and that support has a cost, so it is perhaps not surprising that Len Brown [would like to remove that support](<https://lwn.net/Articles/435688/>) as of 2.6.40. 

Removal of APM support on that schedule is almost certainly not going to happen; a number of developers have expressed concerns that there may still be hardware out there in use which would then be unable to run new kernels. In general, the Linux kernel tries not to abandon users running older hardware. So APM may stay for a while, but there is a problem: keeping APM support, it seems, conflicts with some needed changes to the cpuidle code. The need to keep APM working, in other words, threatens to hold back improvements for the majority of users who have more current hardware. 

The solution to this conflict may take the form of a partial removal of APM support. The most important APM feature for users of old systems is likely to be the ability to power-off the system; other features may be less important. As Andi Kleen [noted](<https://lwn.net/Articles/436050/>), idle support probably matters less to such users: 

Phasing out APM idle at least would be reasonable. Presumably even if the old laptops still work they are likely on AC because their batteries have long died. So using a bit more power in idle shouldn't be a big issue. 

So APM support, as such, may stick around for a while, but it may begin to lose features as the kernel moves on.
