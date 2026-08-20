---
title: CFS bandwidth control
url: https://lwn.net/Articles/428230/
date: "February 16, 2011"
category: "Group scheduling; Scheduler-Completely fair scheduler"
author: "By Jonathan Corbet February 16, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 16, 2011

The CFS scheduler does its best to divide the available CPU time between contending processes, keeping the CPU utilization of each about the same. The scheduler will not, however, insist on equal utilization when there is free CPU time available; rather than let the CPU go idle, the scheduler will give any left-over time to processes which can make use of it. This approach makes sense; there is little point in throttling runnable processes when nobody else wants the CPU anyway. 

Except that, sometimes, that's exactly what a system administrator may want to do. Limiting the maximum share of CPU time that a process (or group of processes) may consume can be desirable if those processes belong to a customer who has only paid for a certain amount of CPU time or in situations where it is necessary to provide strict resource-use isolation between processes. The CFS scheduler cannot limit CPU use in that manner, but the [CFS bandwidth control](<https://lwn.net/Articles/428175/>) patches, posted by Paul Turner, may change that situation. 

This patch adds a couple of new control files to the CPU control group mechanism: `cpu.cfs_period_us` defines the period over which the group's CPU usage is to be regulated, and `cpu.cfs_quota_us` controls how much CPU time is available to the group over that period. With these two knobs, the administrator can easily limit a group to a certain amount of CPU time and also control the granularity with which that limit is enforced. 

Paul's patch is not the only one aimed at solving this problem; the [CFS hard limits patch set](<https://lwn.net/Articles/368685/>) from Bharata B Rao provides nearly identical functionality. The implementation is different, though; the hard limits patch tries to reuse some of the bandwidth-limiting code from the realtime scheduler to impose the limits. Paul has expressed concerns about the overhead of using this code and how well it will work in situations where the CPU is almost fully subscribed. These concerns appear to have carried the day - there has not been a hard limits patch posted since early 2010. So the CFS bandwidth control patches look like the form this functionality will take in the mainline.
