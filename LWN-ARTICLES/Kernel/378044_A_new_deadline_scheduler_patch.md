---
title: A new deadline scheduler patch
url: https://lwn.net/Articles/378044/
date: "March 10, 2010"
category: "Realtime-Deadline scheduling; Scheduler-Deadline scheduling"
author: "By Jonathan Corbet March 10, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 10, 2010

The POSIX approach to realtime scheduling is based on priorities: the highest-priority task gets the CPU. The research community has long since moved on from priorities, though, and has been putting a lot of effort into deadline scheduling instead. Deadline schedulers allow each process to provide a "worst case execution time" and the deadline by which it must get that time; it can then schedule all tasks so that they meet their deadlines while refusing tasks which would cause that promise to be broken. There are a few deadline scheduler patches in circulation, but the `SCHED_DEADLINE` patch by Dario Faggioli and friends looks like the most likely one to make it into the mainline at this time; LWN [looked at this patch](<http://lwn.net/Articles/356576/>) back in October. 

Recently, [version 2 of the `SCHED_DEADLINE` patch](<http://lwn.net/Articles/376502/>) was posted. The changes reflect a number of comments which were made the first time around; among other things, there is a new implementation of the group scheduling mechanism. Perhaps most significant in this patch, though, is an early attempt at addressing priority inversion problems, where a low-priority process can, by holding shared resources, prevent a higher-priority process from running. Priority inversion is a hard problem, and, in the deadline scheduling area, it remains without a definitive solution. 

In classic realtime scheduling, priority inversion is usually addressed by raising the priority of a process which is holding a resource required by a higher-priority process. But there are no priorities in deadline scheduling, so a variant of this approach is required. The new patch works by "deadline inheritance" - if a process holds a resource required by another process which has a tighter deadline, the holding process has its deadline shortened until the resource is released. It is also necessary to exempt the process from bandwidth throttling (exclusion from the CPU when the stated execution time is exceeded) during this time. That, in turn, could lead to the CPU being oversubscribed - something deadline schedulers are supposed to prevent - but the size of the problem is expected to be small. 

The "to do" list for this patch still has a number of entries, including less disruptive bandwidth throttling, a port to the realtime preemption tree, truly global deadline scheduling on multiprocessor systems (another hard problem), and more. The code is progressing, though, and Linux can be expected to have a proper deadline scheduler at some point in the not-too-distant future - though no deadline can be given as the worst case development time is still unknown.
