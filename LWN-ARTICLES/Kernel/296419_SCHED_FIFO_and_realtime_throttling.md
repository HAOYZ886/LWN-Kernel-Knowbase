---
title: SCHED_FIFO and realtime throttling
url: https://lwn.net/Articles/296419/
date: "September 1, 2008"
category: "Realtime; Scheduler-Realtime"
author: "By Jonathan Corbet September 1, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 1, 2008

The SCHED_FIFO scheduling class is a longstanding, POSIX-specified realtime feature. Processes in this class are given the CPU for as long as they want it, subject only to the needs of higher-priority realtime processes. If there are two SCHED_FIFO processes with the same priority contending for the CPU, the process which is currently running will continue to do so until it decides to give the processor up. SCHED_FIFO is thus useful for realtime applications where one wants to know, with great assurance, that the highest-priority process on the system will have full access to the processor for as long as it needs it. 

One of the many features merged back in the 2.6.25 cycle was realtime group scheduling. As a way of balancing CPU usage between competing groups of processes, each of which can be running realtime tasks, the group scheduler introduced the concept of "realtime bandwidth," or rt_bandwith. This bandwidth consists of a pair of values: a CPU time accounting period, and the amount of CPU that the group is allowed to use - at realtime priority - during that period. Once a SCHED_FIFO task causes a group to exceed its rt_bandwidth, it will be pushed out of the processor whether it wants to go or not. 

This feature is required if one wants to allow multiple groups to split a system's realtime processing power. But it also turns out to have its uses in the default situation, where all processes on the system are contained within a single, default group. Kernels shipped since 2.6.25 have set the rt_bandwidth value for the default group to be 0.95 out of every 1.0 seconds. In other words, the group scheduler is configured, by default, to reserve 5% of the CPU for non-SCHED_FIFO tasks. 

It seems that nobody really noticed this feature until mid-August, when Peter Zijlstra posted [a patch](<https://lwn.net/Articles/296421/>) which set the default value to "unlimited." At that point it became clear that some developers have a different idea about how this kind of policy should be set than others do. 

Ingo Molnar [disagreed](<https://lwn.net/Articles/296422/>) with the patch, saying: 

The thing is, i got far more bugreports about locked up RT tasks where the lockup was unintentional, than real bugreports about anyone _intending_ for the whole box to come to a grinding halt because a high-prio RT tasks is monopolizing the CPU. 

Ingo's suggestion was to raise the limit to ten seconds of CPU time. As he (and others) pointed out: any SCHED_FIFO application which needs to monopolize the CPU for that long has serious problems and needs to be fixed. 

There are real problems associated with letting a SCHED_FIFO process run indefinitely. Should that process never get around to relinquishing the CPU, the system will simply hang forevermore; there is no possibility of the administrator slipping in with a `kill` command. This process will also block important things like kernel threads; even if it releases the processor after ten seconds, it will have seriously degraded the operation of the rest of the system. Even on a multiprocessor system, there will typically be processes bound to the CPU where the SCHED_FIFO process is running; there will be no way to recover those processes without breaking their CPU affinity, which is not a step anybody wants to take. 

So, it is argued, the rt_bandwidth limit is an important safety breaker. With it in place, even a runaway SCHED_FIFO cannot prevent the administrator from (eventually) regaining control of the system and figuring out what is going on. In exchange for this safety, this feature only robs SCHED_FIFO tasks of a small amount of CPU time - the equivalent of running the application on a slightly weaker processor. 

Those opposed to the default rt_bandwidth limit cite two main points: it is a user-space API change (which also breaks POSIX compliance) and represents an imposition of policy by the kernel. On the first point, Nick Piggin [worries](<https://lwn.net/Articles/296425/>) that this change could lead to broken applications: 

It's not common sense to change this. It would be perfectly valid to engineer a realtime process that uses a peak of say 90% of the CPU with a 10% margin for safety and other services. Now they only have 5%. 

Or a realtime app could definitely use the CPU adaptively up to 100% but still unable to tolerate an unexpected preemption. 

What could make the problem worse is that the throttle might not cut in during testing; it could, instead, wait until something unexpected comes up in a production system. Needless to say, that is a prospect which can prove scary for people who create and deploy this kind of system. 

The "policy in the kernel" argument was mostly [shot down by Linus](<https://lwn.net/Articles/296427/>), who pointed out that there's lots of policy in the kernel, especially when it comes to the default settings of tunable parameters. He says: 

And the default policy should generally be the one that makes sense for most people. Quite frankly, if it's an issue where all normal distros would basically be expected to set a value, then that value should _be_ the default policy, and none of the normal distros should ever need to worry. 

Linus carefully avoided taking a position on which setting makes sense for the most people here. One could certainly argue that making systems resistant to being taken over by runaway realtime processes is the more sensible setting, especially considering that there is a certain amount of interest in running scary applications like PulseAudio with realtime priority. On the other hand, one can also make the case that conforming to the standard (and expected) SCHED_FIFO semantics is the only option which makes sense at all. 

There has been some talk of creating a new realtime scheduling class with throttling being explicitly part of its semantics; this class could, with a suitably low limit, even be made available to unprivileged processes. Meanwhile, as of this writing, the 0.95-second limit - the one option that _nobody_ seems to like - remains unchanged. It will almost certainly be raised; how much is something we'll have to wait to see.
