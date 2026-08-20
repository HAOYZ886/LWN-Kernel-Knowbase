---
title: The burstable CFS bandwidth controller
url: https://lwn.net/Articles/844976/
date: "February 8, 2021"
category: "Control groups; Scheduler-Completely fair scheduler"
author: "By Jonathan Corbet February 8, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 8, 2021

The kernel's CFS bandwidth controller is an effective way of controlling just how much CPU time is available to each control group. It can keep processes from consuming too much CPU time and ensure that adequate time is available for all processes that need it. That said, it's not entirely surprising that the bandwidth controller is not perfect for every workload out there. [This patch set](<https://lwn.net/ml/linux-kernel/20210202114038.64870-1-changhuaixin@linux.alibaba.com/>) from Huaixin Chang aims to make it work better for bursty, latency-sensitive workloads. 

The bandwidth controller only applies to "completely fair scheduling" (CFS) tasks (otherwise known as "normal processes"); the CPU usage of realtime tasks is handled by other means. This controller provides two parameters to manage the limits applied to any given control group: 

  * `cpu.cfs_quota_us` is the amount of CPU time (in microseconds) available to the group during each accounting period. 
  * `cpu.cfs_period_us` is the length of the accounting period, also in microseconds. 

Thus, for example, setting `cpu.cfs_quota_us` to 50000 and `cpu.cfs_period_us` to 100000 will enable the group to consume 50ms of CPU time in every 100ms period. Halving those values (setting `cpu.cfs_quota_us` to 25000 and `cpu.cfs_period_us` 50000) allows 25ms of CPU time every 50ms. In both cases, the group has been empowered to consume 50% of one CPU, but in the latter case that time will come more frequently, in smaller chunks. 

The distinction between those two cases is important here. Imagine a control group containing a single process that needs to run for 30ms. In the first case, 30ms is less than the allowed 50ms, so the process will be able to complete its task without being throttled. In the second case, the process will be cut off after running for 25ms; it will then have to wait for the next 50ms period to start before it can finish its job. If the workload is sensitive to latency, the bandwidth-controller parameters need to be set with care. 

This mechanism works reasonably well for workloads that consistently require a specific amount of CPU time. It can be a bit more awkward, though, for bursty workloads. A given process may use far less than its quota during most periods, but occasionally a burst of work may come along that requires more CPU time than the quota allows. In cases where latency doesn't matter, making that process wait for the next period to finish its work may not be a problem; if latency does matter, though, this delay can be a real concern. 

There are ways to try to work around this issue. One, of course, is to just give the process in question a quota that is large enough to handle the workload bursts, but doing that will enable the process to consume more CPU time overall. System administrators may not like that result, especially if there is money involved and only so much time is actually being paid for. The alternative would be to increase both the quota and the period, but that, too, can increase latency if the process ends up waiting for the next period anyway. 

Chang's patch set enables a different approach: allow control groups to carry over some of their unused quota from one period to the next. A new parameter, `cpu.cfs_burst_us`, sets the maximum amount of time that can be accumulated that way. As an example, let's return to the group with a quota of 25ms and a period of 50ms. If `cpu.cfs_burst_us` is set to 40000 (40ms), then processes in that group can run for up to 40ms in a given period, but only if they have carried over the 15ms beyond their normal quota from previous periods. This allows the group to respond to a burst of work while still keeping it within the quota in the longer term. 

Another way of looking at this situation is that, when `cpu.cfs_burst_us` is in use, the quota is interpreted differently than before. Rather than being an absolute limit, the quota is an amount of CPU time that is deposited into the group's CPU-time account every period, with the burst value capping the value of that account. Bursty groups can save up a limited amount of CPU time in that account for when they need it. 

By default, `cpu.cfs_burst_us` is zero, which disables the burst mechanism and preserves the previous behavior. There is a sysctl knob that can be used to disable burst usage across the entire system. Another knob (`sysctl_sched_cfs_bw_burst_onset_percent`) causes the controller to give each group a given percentage of their burst quota at the beginning of each period, regardless of whether that time has been accumulated in previous periods. 

The patch set comes with some benchmark results showing order-of-magnitude reductions in worst-case latencies when the burstable controller is in use. This idea has been seen on the lists a few times at this point, both in its current form and as separate implementations by [Cong Wang](<https://lore.kernel.org/lkml/20180522062017.5193-1-xiyou.wangcong@gmail.com/>) and [Konstantin Khlebnikov](<https://lore.kernel.org/lkml/157476581065.5793.4518979877345136813.stgit@buzz/>). It looks as if the biggest roadblocks have been overcome at this point, so this change could find its way into the mainline as soon as the 5.13 merge window.
