---
title: "Toward better CPU idle-time predictions"
url: https://lwn.net/Articles/618074/
date: "October 29, 2014"
category: "Power management; Scheduler-and power management"
author: "By Jonathan Corbet October 29, 2014 Linux
Plumbers Conference"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
October 29, 2014

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2014-Linux_Plumbers_Conference>)

The CPU idle ("cpuidle") code has one of those tasks that would be best handled with absolute knowledge of the future: knowing how long the processor will be idle so that the most appropriate sleep state can be chosen. Since that knowledge is hard to come by, the cpuidle code must get by with heuristics. At the 2014 Linux Plumbers Conference (LPC), Daniel Lezcano talked about a scheme he has to improve those heuristics and, in the process, bring about better integration between the scheduler and the cpuidle subsystem. 

The current cpuidle code suffers from a number of shortcomings. It is not actually tied to the scheduler, so it has no idea of what the scheduler plans to do next. Even the most sophisticated governors tend to get things wrong, leading to the wrong sleep states being chosen. By focusing on a few relatively predictable parameters, Daniel hopes to come up with an approach to cpuidle that is both simpler and more accurate. 

The "menu" cpuidle governor used on many systems looks at the recent past and tries to come up with a good guess as to how long the system will sleep the next time it goes idle. But actual system behavior depends on a wide variety of different events. Some, like timer expirations, are entirely predictable. Others, such as block I/O operations, are reasonably [![\[Daniel Lezcano\]](https://static.lwn.net/images/conf/2014/lce-lpc/DanielLezcano-sm.jpg)](<https://lwn.net/Articles/618125/>) predictable, especially if one is watching carefully. Others, including things like keyboard events, are not predictable at all. Including these latter events in the calculation, Daniel said, leads to bad predictions and erratic performance from the cpuidle governor. 

Daniel's [cpuidle patch set](<https://lwn.net/Articles/617598/>) addresses the most predictable wakeup events by having the scheduler pass the next timer expiration time into the cpuidle code. The scheduler also passes in the current latency requirement; cpuidle can use that information to avoid putting the processor into an overly deep sleep. The most unpredictable events, instead, are simply ignored in this version of the cpuidle code. 

That leaves the moderately predictable events, primarily block I/O. Daniel's patch set starts by maintaining a simple running average of per-task I/O completion times. All tasks waiting for block I/O on a given CPU are put into a [red-black tree](<https://lwn.net/Articles/184495/>); the closest expected completion time is easily obtained from that tree and used to predict the next wakeup time. But the running average is a bit too simple, being overly affected by the occasional operation that takes much longer (or much less time) than expected. So something a bit more complex is called for. 

Daniel's response is to divide I/O completion times into buckets; after some investigation, he settled on 200µs as the optimal bucket granularity. Each bucket contains a counter of "hits," being the number of times that an operation has fallen into that bucket's duration. The buckets tracking I/O completion times are stored in a linked list. When a process first starts up, the data structure might look something like this: 

> ![\[cpuidle bucket data structure\]](https://static.lwn.net/images/2014/cpuidle-buckets.png)

The "hits" counter is incremented in the appropriate bucket for each I/O operation completion. After a small number of operations, the data structure might look like this: 

> ![\[cpuidle bucket data structure\]](https://static.lwn.net/images/2014/cpuidle-buckets2.png)

Something interesting happens every time one bucket gains five hits though: it is moved to the beginning of the list. So, if the next operation completes in just over 200µs, the data structure will look like this: 

> ![\[cpuidle bucket data structure\]](https://static.lwn.net/images/2014/cpuidle-buckets3.png)

The idea is that the buckets that see the most activity will be found toward the beginning of the list. When it comes time to predict when the next operation will complete, Daniel's code iterates through the list, computing a score for each bucket. Essentially, that score is the number of hits in the bucket divided by 1+2 _p_ , where _p_ is the bucket's position in the list. So the bucket in the second position must have three times as many hits to get a higher score than the bucket in the first position. 

The idea is to try to guess the most likely completion time based on both long-term and recent history. According to Daniel, it works pretty well, yielding far better results than the existing menu governor. Even so, this design did not survive the discussion in the LPC microconference, so any version of this patch set that gets into the mainline kernel is likely to look somewhat different. 

The issue was the use of a per-task data structure for I/O completion time tracking. The advantage of this approach is that, when a task moves between CPUs, the tracking information will move with it, so the scheduler on the new CPU has an immediate idea of how long that task's I/O operations will take. But I/O completion times are not really a task-specific parameter; they are, instead, tied to the underlying device. And, as it happens, the kernel is already tracking device performance in the block layer. That information reflects current loads and should be quite accurate; using it might enable the entire bucket data structure to be done without. 

So a future version of this patch set will probably be recast along those lines, using the backing store information already maintained by the kernel. But there are other challenges looming for this code. As Peter Zijlstra pointed out, the block layer is increasingly trying to maintain locality in I/O requests, ensuring that the CPU handling an I/O completion is the one that initiated the operation. Better locality makes sense, Peter said, but it also can conflict with the scheduler's attempts at distributing load across a system. It is harder to guess at wakeup times if it's not clear which CPU will wake up to deal with a specific I/O completion. The multiqueue block layer code is going to make this problem worse; some work will have to be done to reconcile these differing approaches to performance. 

But, even if it is not a complete solution to the problem, Daniel's patch achieves the goals of better sleep-time predictions and better integration of the cpuidle code with the scheduler. That should be sufficient to get some version of this code into the mainline — someday. 

[Your editor would like to thank the Linux Foundation for supporting his travel to this event].
