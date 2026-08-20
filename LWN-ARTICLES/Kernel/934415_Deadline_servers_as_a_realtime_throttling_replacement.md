---
title: Deadline servers as a realtime throttling replacement
url: https://lwn.net/Articles/934415/
date: "June 12, 2023"
category: "Releases-6.8; Scheduler-Realtime"
author: "By Jonathan Corbet June 12, 2023"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
June 12, 2023

The CPU scheduler's one job at any given time is to run the task that has the strongest claim to the CPU. There are many factors that complicate that job, not the least of which is that the "strongest claim" is sometimes a bit of a fuzzy concept. Realtime throttling, a mechanism designed to keep a runaway realtime task from monopolizing the CPU, is one case where developers have concluded that the task with, ostensibly, the highest priority should not actually be the one that runs. But realtime throttling has rarely pleased anybody; the [deadline-server infrastructure patches](<https://lwn.net/ml/linux-kernel/cover.1686239016.git.bristot@kernel.org/>) posted by Daniel Bristot de Oliveira are the latest attempt to find a better solution. 

The POSIX realtime scheduling classes are conceptually simple; at any given time, the task with the highest priority runs to the exclusion of anything else. In the real world, though, the rule enables a runaway realtime task to take over the system to the point that the only way to recover it may be to pull the plug. Power failures, as it turns out, have an even higher priority than realtime tasks. 

Yanking out the power cord is aesthetically displeasing to many, though, and tends to cause realtime deadlines to be missed; in an attempt to avoid it, the kernel developers introduced realtime throttling many years ago. In short, realtime throttling restricts realtime tasks to (by default) 95% of the available CPU time; the remaining 5% is left for lower-priority tasks, with the idea that it is enough for an administrator to kill off a runaway task if need be. 

Most of the time, this throttling is not a problem. In a properly designed realtime system, the actual realtime work should be using far less than 95% of the available CPU time anyway, so the throttling will never actually happen. But, in cases where a realtime task does need all of the available CPU time for an extended period, realtime throttling can be a problem. This is especially true because the throttling happens even if there are no lower-priority tasks waiting to run. Rather than run the realtime task that still needs CPU, the scheduler will simply force the system idle in this case. The idle time is an unwanted artifact of how the throttling is implemented rather than a desired feature in its own right. 

Various efforts have been made to address this problem over the years; [this article](<https://lwn.net/Articles/931789/>) describes one approach, where realtime throttling would be disabled if it would cause the system to go idle. The deadline-server idea is a different approach to the problem, based on the [deadline scheduling](<https://lwn.net/Articles/743740/>) class. This class, which has a higher priority than the POSIX realtime classes, is not priority-based; instead, tasks declare the amount of CPU time they need and the time by which they must receive it, and the deadline scheduler works to ensure that those tasks meet their deadlines. 

This class thus seems like a natural way to take back 5% of the CPU from realtime tasks when needed. All that is needed is to create a task in the deadline class (called the "deadline server"), declare that it needs 5% of the CPU, and have that task run lower-priority tasks with the time that it is given. The scheduler will then carve out the necessary CPU time but, if the deadline server doesn't need it, it will simply not be runnable and the realtime tasks can continue to run. 

The idea, as implemented in Bristot's patch set (which contains patches from Peter Zijlstra and Juri Lelli), does the job reasonably well, in that it makes space for lower-priority tasks without needlessly causing the CPU to go idle. The fact that the deadline class has a higher priority than the realtime classes makes this idea work, but also brings one little problem: once the deadline server is enabled, it will run immediately, perhaps preempting a realtime task that would have eventually yielded anyway. The lower-priority tasks should get their 5%, but giving it to them immediately may create problems for well-behaved realtime tasks. 

The proposed solution here is to delay the enabling of the deadline server. A kernel timer is used to occasionally run a watchdog function that looks at the state of the normal-priority tasks on the system. If it appears that those tasks are being starved — with starvation defined as not getting any CPU time over a half-second — then the deadline server will be started. Otherwise, in the absence of starvation problems, scheduling will run as usual. 

With this tweak, the work is moving ""in the right direction"", Bristot said, but there is still room for improvement. The delay of the startup of the deadline server can be further delayed to the "zero-laxity" time — the time just before it would miss a 5% deadline entirely. The starvation monitor could perhaps be moved to CPUs that are not running realtime tasks to prevent interference there. In general, though, this work looks like it could be a plausible solution to the realtime-throttling problem.
