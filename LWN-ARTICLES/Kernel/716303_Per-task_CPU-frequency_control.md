---
title: "Per-task CPU-frequency control"
url: https://lwn.net/Articles/716303/
date: "March 8, 2017"
category: "Scheduler-and power management"
author: "By Jonathan Corbet March 8, 2017"
---

By **Jonathan Corbet**  
March 8, 2017

The kernel's power-management code attempts to run each processor on the system at a level that minimizes power consumption while ensuring that sufficient CPU time is available for the currently running tasks. CPU frequency management has, over the last few years, [become more closely tied to the scheduler](<https://lwn.net/Articles/552885/>), since that is where the information about the current workload resides. The scheduler, however, does not know which processes are most important to the user. Various attempts to fill in that information have been made over time, with none making it into the mainline; the latest version takes a different approach. 

The core idea behind workload-sensitive power management is that the user (or, more likely, some sort of policy daemon working on the user's behalf) may want to influence how decisions are made depending on which processes are running. For processes that the user would like to see run quickly — those currently running in the foreground on a handset, for example — it may be desirable to run the CPU at a higher rate than is strictly necessary to get the expected amount of work done. On the other hand, if only a low-priority background task is running, it may be best to put an upper limit on how fast the CPU runs, even if that task has a lot of work to do. At the moment, however, the power-management code cannot distinguish those types of process from each other, so the same frequency-scaling policies apply to all of them. 

Recent attempts to solve this problem have taken the form of a control-group controller called [SchedTune](<https://lwn.net/Articles/706374/>). This controller allowed a "boost" value to be applied to processes in a specific control group. Those processes would be made to appear to require more CPU time than they actually needed, causing the CPU-frequency governor to pick a higher frequency than it otherwise would have. This approach worked, but one might argue that the approach of distorting the apparent load to influence frequency selection lacked elegance. 

At the end of February, Patrick Bellasi posted [a new patch set](<https://lwn.net/Articles/715869/>) that takes a different approach. The separate SchedTune controller is no more; instead, CPU-frequency policy has been moved into the core CPU controller, where it can be found alongside the other scheduling parameters for any given control group. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The "boost" value and the load-distorting algorithm it used are gone. In their place are two new control knobs, called `capacity_min` and `capacity_max`. They place bounds on the CPU frequency choices that can be made when any process in the group is running. The `capacity_min` value describes the slowest allowable CPU speed; by default, it is set to zero, meaning that even the slowest CPU frequency is acceptable. The maximum allowable frequency is set by `capacity_max`; the default value here is 1024, allowing the CPU to go to its maximum speed. An important process can thus be guaranteed a certain minimum CPU performance by setting `capacity_min` to an appropriate value, while low-priority tasks can be prevented from pushing the CPU frequency too high with `capacity_max`. 

At any given time, there may be multiple runnable processes, and they may not all have the same `capacity_min` and `capacity_max` parameters. Changing the CPU's operating parameters is a relatively expensive operation, so it does not make sense to change the operating frequency every time a new process is given access to the CPU. One could also argue that, when a process with relatively high CPU-power requirements is waiting, the other processes should be run at just as high a power level to avoid delaying that process excessively. 

The end result is that the scheduler needs to pick a set of parameters that is suitable for all of the processes that are currently runnable. To meet that requirement, the controller will apply the maximum value of both parameters. That ensures that the process(es) with the highest values will actually get those values, and no process will run at a lower CPU frequency than it is entitled to. Implementing this policy requires adding two [red-black trees](<https://lwn.net/Articles/184495/>) to each control group tracking the processes with the highest `capacity_min` and `capacity_max` values. 

When multiple levels of control groups are in use, subgroups are only allowed to tighten the constraints set in their parent groups. So `capacity_min` in a subgroup cannot go below that value in the parent, while `capacity_max` cannot exceed the parent's value. 

In previous patch sets, this feature has been focused on the `SCHED_OTHER` (normal) scheduling class. With this patch set, though, it has also been extended to the realtime and deadline scheduling classes. In current kernels, those classes are run at the maximum speed the processor supports. With this change, realtime and deadline scheduling can be used in a more power-friendly mode. Needless to say, tuning of these parameters with such workloads will need to be done carefully to avoid configuring a system that cannot meet its realtime requirements. 

As of this writing, there have been no comments on the new patch set. That, perhaps, is one of the hazards of posting core-kernel patches during the merge window. One might guess that this version offers relatively little to complain about, but experience suggests that one might easily guess incorrectly when it comes to scheduler patches. Once the scheduler developers have a chance to look at this code, we'll have a better idea of whether it's likely to get into the mainline in its current form.
