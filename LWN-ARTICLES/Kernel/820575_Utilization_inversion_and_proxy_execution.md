---
title: Utilization inversion and proxy execution
url: https://lwn.net/Articles/820575/
date: "May 15, 2020"
category: "Scheduler-Load tracking; Scheduler-Proxy execution"
author: "By Jonathan Corbet May 15, 2020 OSPM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 15, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

Over the years, the kernel's CPU scheduler has become increasingly aware of how much load every task is putting on the system; this information is used to make smarter task placement decisions. Sometimes, though, this logic can go wrong, leading to a situation that Valentin Schneider describes as "utilization inversion". At the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM), he described the problem and some approaches that are being considered to address it. 

Utilization tracking, initially conceived as [per-entity load-tracking](<https://lwn.net/Articles/531853/>) or PELT, gives the scheduler information about what the impact of running a task on a given CPU will be. It is used in the schedutil CPU-frequency governor to select a frequency appropriate to the current workload. On Arm [big.LITTLE](<https://en.wikipedia.org/wiki/ARM_big.LITTLE>) systems, where some processors are faster and more power-hungry than others, the utilization-tracking signal is also used to decide which type of CPU a task should run on. The situation is complicated a bit by [utilization clamping](<https://lwn.net/Articles/762043/>), which allows an administrator to make a task's utilization appear larger or smaller than it really is. Clamping is used to bias the placement of specific tasks and influences CPU-frequency selection as well. 

Imagine, Schneider said, a large task (one with a high utilization) and a small task, both of which are contending for the same lock. The large task may have a high minimum clamp, so it looks like an even bigger load even when it is not doing much; the small task, instead, may have a low maximum, ensuring that it always looks small. One would expect the large task to [![\[Valentin
Schneider\]](https://static.lwn.net/images/conf/2020/ospm/Lelli-Schneider-sm.png)](<https://lwn.net/Articles/820586/>) run on a big CPU at a high frequency while the small task is consigned to a small CPU at a low frequency. If the small task grabs the lock, the large task's progress suddenly depends on how quickly the small task can progress. 

This situation is similar to priority inversion, though the problem is not as severe. Even so, it would be better if the small task could inherit some of the large task's resources while it holds the lock. 

The kernel's realtime mutexes can handle priority inheritance now; if a high-priority task contends for a lock held by a low-priority task, the latter will have its priority boosted until it drops the lock. Priority inheritance can help, but it only affects process priority; it can force preemption, but it does not really change task placement or CPU frequency. Perhaps the kernel could gain a similar mechanism for utilization that would help for placement, at least, if not CPU frequency. Schneider expressed skepticism that such an approach could work well, though. 

An alternative he has been working on is proxy execution: giving the lock-holding task the waiting task's execution parameters until it lets go of the lock. This is a work in progress, he said, that doesn't survive for more than 20 seconds on real hardware, and it has no provision for futexes (user-space locks), but it still has some interesting properties, he said. 

With proxy execution, a task that blocks on a mutex is not removed from the run queue as it would be in a mainline kernel. It can thus be picked to run by the scheduler in the usual way if it's the highest-priority task in the queue. When that happens, though, the lock-holding task inherits the blocked task's scheduling context. The blocked task is also migrated to the run queue of the lock holder, which brings its utilization information over; that will cause the CPU frequency to be increased, helping the lock holder to get its work done and release the lock. 

That solves the problem reasonably well on _symmetric_ multiprocessor systems, but it still falls short on asymmetric systems like big.LITTLE. To address such systems, Schneider would like to put the utilization-tracking information into the scheduling context, where it can be passed more directly to a lock holder. This has to be done carefully, though, or it could create priority inversions of its own; if a low-utilization task is picked to run, it could end up slowing a high-utilization task. Making a smart choice is hard, though, since the utilization signals are highly variable and hard to track in the proxy-execution code. The solution might be to ignore the utilization values and just look at the clamps. 

Juri Lelli asked why this mattered, since the clamp values are already aggregated on each run queue. That works for frequency selection, Schneider answered, but it has no influence on task selection, so it doesn't help to ensure that the lock-holding task actually runs. 

Then, there is the perennial problem of load balancing. Utilization signals are highly useful here, since they let the scheduler ensure that the load on each CPU is about the same. But what should be done in the proxy execution case? Currently, load-balancing decisions will use the scheduling context of the donor task (the one waiting for the lock), which could lead to interesting decisions. Since contending tasks remain on the run queue, the apparent load on the CPU increases, which can throw things off as well. Peter Zijlstra said that this isn't necessarily a big problem; one does not expect locks to be held for long periods, so things should straighten themselves out relatively quickly. 

Patrick Bellasi asked whether just relying on clamp values is sufficient, or whether the load-tracking signal should be used too. Schneider responded that using the clamps really is the best that can be done; there is no choice. Utilization values simply change too quickly to be useful. 

Heading toward a conclusion, Schneider said that getting proxy execution working right is his first priority; presumably rebooting after 20 seconds of uptime is getting a little tiresome. He asked whether other developers were interested in proxy execution as well. Zijlstra said that he has been trying to get it to the top of his list for a long time, but has been "failing miserably". 

Qais Youssef asked how quickly this work might be done. The next Android release will not be happening for some time, so it would be nice if there were some way to fix this problem in the short term. Could the realtime mutex code help? Zijlstra responded that realtime mutexes are really for realtime processes and won't help with tasks in the completely fair scheduling class, as most Android tasks are. We will get the problem solved when we do, he said. 

The session concluded with numerous developers saying that they would like to have a working proxy execution mechanism in the kernel, but nobody has found the time to work on it.
