---
title: "Is the whole system idle?"
url: https://lwn.net/Articles/558284/
date: "July 10, 2013"
category: "Dynamic tick; Read-copy-update"
author: "By Jonathan Corbet July 10, 2013"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 10, 2013

The [full dynamic tick](<https://lwn.net/Articles/549580/>) feature that made its debut in the 3.10 kernel can be good for users who want their applications to have full use of one or more CPUs without interference from the kernel. By getting the clock tick out of the way, this feature minimizes kernel overhead and the potential latency problems. Unfortunately, full dynamic tick operation also has the potential to increase power consumption. Work is underway to fix that problem, but it turns out to require a bit of information that is surprisingly hard to get: is the system fully idle or not? 

The kernel has had the ability to turn off the periodic clock interrupt on idle processors for many years. Each processor, when it goes idle, will simply stop its timer tick; when all processors are idle, the system will naturally have the timer tick disabled systemwide. Fully dynamic tick — where the timer tick can be disabled on non-idle CPUs — adds an interesting complication, though. While most processors can (when the conditions are right) run without the clock tick, one processor must continue to keep the tick enabled so that it can perform a number of necessary system timekeeping operations. Clearly, this "timekeeping CPU" should be able to disable its tick and go idle if nothing else is running in the system, but, in current kernels, there is no way for that CPU to detect this situation. 

A naive solution to this problem will come easily to mind: maintain a global counter tracking the number of idle CPUs. Whenever a processor goes idle, it increments the counter; when the processor becomes busy again, it decrements the counter. When the number of idle CPUs matches the number of CPUs in the system, the kernel will know that no work is being done and the timekeeping CPU can take a break. 

The problem, of course, is that cache contention for that global counter would kill performance on larger systems. Transitions to and from idle are common under most workloads, so the cache line containing the counter would bounce frequently across the system. That would defeat some of the point of the dynamic tick feature; it seems likely that many users would prefer the current power-inefficient mode to a "solution" that carried such a heavy cost. 

So something smarter needs to be done. That's the cue for an entry by Paul McKenney, whose [seven-part full-system idle patch set](<https://lwn.net/Articles/558229/>) may well be the solution to this problem. 

As one might expect, the solution involves the maintenance of a per-CPU array of idle states. Each CPU can update its status in the array without contending with the other CPUs. But, once again, the naive solution is inadequate. With a per-CPU array, determining whether the system is fully idle requires iterating through the entire array to examine the state of each CPU. So, while maintaining the state becomes cheap, answering the "is the system idle?" question becomes expensive if the number of CPUs is large. Given that the timekeeping code is likely to want to ask that question frequently (at each timer tick, at least), an expensive implementation is not indicated; something else must be done. 

Paul's approach is to combine the better parts of both naive solutions. A single global variable is created to represent the system's idle state and make that state easy to query quickly. That variable is updated from a scan over the individual CPU idle states, but only under specific conditions that minimize cross-CPU contention. The result should be the best of both worlds, at the cost of delayed detection of the full-system idle state and the addition of some tricky code. 

The actual scan of the per-CPU idle flags is not done in the scheduler or timekeeping code, as one might expect. Instead (as others might expect), Paul put it into the read-copy-update (RCU) subsystem. That may seem like a strange place, but it makes a certain sense: RCU is already tracking the state of the system's CPUs, looking for "grace periods" during which unused RCU-protected data structures can be reclaimed. Tracking whether each CPU is fully idle is a relatively small change to the RCU code. As an added benefit, it is easy for RCU to avoid scanning over the CPUs when things are busy, so the overhead of maintaining the global full-idle state vanishes when the system has other things to do. 

The actual idleness of the system is tracked in a global variable called `full_sysidle_state`. Updating this variable too often would bring back the cache-line contention problem, though, so the code takes a more roundabout path. Whenever the system is perceived to be idle, the code keeps track of when the last processor went idle. Only after a delay will the global idle state be changed. That delay drops to zero for "small" machines (those with no more than eight processors), it increases linearly as the number of processors goes up. So, on a very large system, all processors must be idle for quite some time before `full_sysidle_state` will change to reflect that state of affairs. 

The result is that detection of full-system idle will be delayed on larger systems, possibly by a significant fraction of a second. So the timer tick will run a little longer than it strictly needs to. That is a cost associated with Paul's approach, as is the fact that his patch set adds some 500 lines of core kernel code for what is, in the end, the maintenance of a single integer value. But that, it seems, is the price that must be paid for scalability in a world where systems have large numbers of CPUs.
