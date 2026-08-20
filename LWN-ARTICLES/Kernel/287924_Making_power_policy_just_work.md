---
title: Making power policy just work
url: https://lwn.net/Articles/287924/
date: "June 30, 2008"
category: Power management
author: "By Jonathan Corbet June 30, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 30, 2008

The `sched_mc_power_savings` parameter (cleverly hidden under `/sys/devices/system/cpu`) was introduced in the 2.6.18 kernel. If this parameter is set to one (the default is zero), it changes the scheduler load balancing code in an interesting way: it makes an ongoing effort to gather together processes on the smallest number of CPUs. If the system is not heavily loaded, this policy will result in some processors being entirely idle; those processors can then be put into a deep sleep and left there for some time. And that, of course, results in lower power consumption, which is a good thing. 

Vaidyanathan Srinivasan recently [noted](<http://lwn.net/Articles/287882/>) that, while this policy works well in a number of situations, there are others where things could be better. The sched_mc_power_savings policy is relatively conservative in how it loads processes onto CPUs, taking care to not overload those CPUs and create excessive latency for applications. As a result, the workload on a large system can still end up spread out more widely than might be optimal, especially if the workload is bursty. In response, Vaidyanathan suggests making the power savings policy more flexible, with the system administrator being able to select a combination of power savings and latency which works well for the workload. On systems where power savings matters a lot, a more aggressive mode (which would pack processes more tightly into CPUs) could be chosen. 

This suggestion was controversial. Nobody disputes the idea that smarter power savings policy would be a good idea. But there is resistance to the idea of creating more tuning knobs to control this policy; instead, it is felt, the kernel should work out the optimal policy on its own. As Andi Kleen [puts it](<https://lwn.net/Articles/287937/>): 

Tunables are basically "we give up, let's push the problem to the user" which is not nice. I suspect a lot of users won't even know if their workloads are bursty or not. Or they might have workloads which are both bursty and not bursty. 

There are a couple of answers to that objection. One is that the system cannot know, on its own, what priorities the users and/or administrators have. Those priorities could even change over time, with performance being emphasized during peak times and low power usage otherwise. Additionally, not all users see "performance" the same way; some want responsiveness and low latency, while others place a higher priority on throughput. If the system cannot simultaneously optimize all of those parameters, it will need guidance from somewhere to choose the best policy. 

And that's where the other answer comes in: that guidance could come from user space. Special-purpose software running on large installations can monitor the performance of important applications and adjust resources (and policies) to get the desired results. Or, in a somewhat different vision, individual applications could register their performance needs and expected behavior. In this case, the kernel is charged with somehow mediating between applications with different expectations and coming up with a reasonable set of policies. 

In the middle of all this, it was pointed out that a mechanism by which expectations can be communicated to the kernel already exists: the nice level (priority) associated with each process. In a simple view of the world, a process's nice level would tell the kernel how to manage it with regard to power savings; on a system with a number of niced processes, those processes would be gathered onto a subset of processors during period of relatively low activity. In essence, this policy says that it is not worthwhile to power up more processors just to give better throughput to low-priority processes. 

It does not take long, though, to come up with situations where the use of nice levels leads to the wrong sort of results. Peter Zijlstra [observed](<https://lwn.net/Articles/287957/>) that he has niced processes (created with distcc) which should have access to all of the CPU power available, but which should not contend with interactive processes on the same system. In such cases, those processes should have a high nice value with regard to CPU usage, but that should not interfere with their ability to move onto idle CPUs, if any exist. So the answer may take the form of a separate "powernice" command which would regulate a process's priority when it comes to causing the system to draw more power. 

Nice levels may (or may not) prove to be sufficient information to let the system choose an optimal power policy. But it will be some time before anybody really knows that; work on optimizing power usage - especially on server systems - is not in an advanced state. So pressure to add tuning knobs for power policies may continue, for one simple reason: people want ways of experimenting with different policies and seeing what the results are. Until we really know what the effects of different policies are - on both power usage and system performance - it will be hard to build a system which can choose an optimal policy on its own.
