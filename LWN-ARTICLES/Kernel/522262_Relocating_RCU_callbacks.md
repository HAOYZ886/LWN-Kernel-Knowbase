---
title: Relocating RCU callbacks
url: https://lwn.net/Articles/522262/
date: "October 31, 2012"
category: "Dynamic tick; Read-copy-update"
author: "By Jonathan Corbet October 31, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 31, 2012

The read-copy-update (RCU) subsystem is one of the kernel's key scalability mechanisms; it is usually invoked in situations where normal locking is far too slow. RCU is known to be complex code, to the point that [lesser kernel developers](<https://lwn.net/Articles/456240/>) will happily proclaim that they do not understand it. That should not be taken to mean that RCU cannot be made faster or more complex, though. Paul McKenney's "callback-free CPUs" patch set is a case in point. 

Much RCU processing has traditionally been done in software interrupt (softirq) context, meaning that the actual processing is done at seemingly random times during the execution of whatever process happens to have the CPU at the time. Softirqs thus have the potential to add arbitrary delays to the execution of any process, regardless of that process's priority. It is not surprising that the realtime developers [have been working](<https://lwn.net/Articles/520076/>) on the softirq problem; non-realtime developers, too, have been known to grumble about softirq overhead. Depending on the load on the system, RCU processing can be a significant part of the overall softirq workload. So improvements in RCU processing can help eliminate unwanted latencies and jitter even if software interrupt handling as a whole remains unchanged. 

Paul recently [described some work in that direction](<https://lwn.net/Articles/518953/>) on this page; as of the 3.6 kernel, much of the RCU grace period handling has been moved to kernel threads. RCU works by replacing a data structure with a modified version, retaining the old copy but hiding it from view so that no new references to it will be created. The RCU rules guarantee that any data structure made inaccessible in this way before a "grace period" passes will have no outstanding references after that period; the determination of grace periods is thus a crucial step in the cleanup and deletion of those old data structures. It turns out that identifying grace periods in a scalable and efficient manner is not a trivial task; see, for example, [this article](<https://lwn.net/Articles/305782/>) for details. 

Moving grace period handling to kernel threads takes a certain amount of RCU overhead out of the softirq path, reducing jitter and allowing that handling to be assigned priorities like any other process. But, even with grace period processing out of the way, RCU still has a fair amount of work to do in softirq context. Near the top of the list is the calling of RCU callbacks — the functions that actually perform cleanup work after a grace period passes. With some workloads, the number of callbacks can get quite large. Users concerned about jitter have expressed a desire to move as much kernel processing out of the way as possible; RCU callback processing represents a significant chunk of that work. 

That is the motivation for Paul's [callback-free CPUs patch set](<https://lwn.net/Articles/522145/>). The idea is simple enough: rather than invoke RCU callbacks in softirq context, the kernel can just shunt that work off to yet another kernel thread. The implementation, of course, is just a bit more involved than that. 

The patch set adds a new `rcu_nocbs=` boot-time parameter allowing the system administrator to specify a set of CPUs to run in the "no callbacks" mode. It is not possible to do so with every CPU in the system; at least one processor must remain in the traditional mode or grace period processing will not function properly. In practical terms, that means that CPU0 cannot be run in the no-callbacks mode and any attempt to hot-remove the last traditional-RCU CPU will fail. 

When a CPU (call it `CPU _N_`) runs without RCU callbacks, there will be a separate `rcuo _N_` process charged with callback handling. When that process wakes up, it will grab the list of outstanding callbacks for its assigned CPU, using some tricky atomic-exchange techniques to avoid the need for explicit locking. The thread will wait for the grace period to expire, then run through the callbacks; after that the cycle begins anew. Normally the process wakes up when callbacks are added to an empty list, but a separate boot parameter instructs the threads to poll occasionally for new work instead. Polling has its costs, especially on systems where energy efficiency and letting CPUs sleep are priorities, but it can improve RCU's CPU efficiency, helping throughput. 

Users who are so sensitive to jitter that they want to reconfigure RCU callback processing may not be satisfied just by having that processing move to a thread that competes with their workload. The good news for those users is that, once callback processing lives in its own thread, it can be assigned a priority that fits with the overall goals of the system. Perhaps even better, the callback thread does not have to run on the CPU whose callbacks it is handling; by playing with CPU affinities, administrators can move that work to other CPUs, freeing the no-callback CPUs to focus more exclusively on the user's workload. 

No-callback CPUs are thus part of the larger effort toward fully-dedicated CPUs that run nothing but the user's processes. The idea is that, on such a CPU, the workload would be fully in charge and need never worry that the kernel would get in the way when there is time-sensitive work to be done. Solving that problem in a robust and maintainable manner is a rather larger problem; it requires the [NoHZ mechanism](<https://lwn.net/Articles/420544/>) and more. It has been recognized for some time that this problem will need to be solved in smaller pieces; the no-callback CPUs patch is one of those pieces. 

This patch set is in its second iteration; comments this time around have been scarce. Barring surprises, it would not be surprising to see this feature pushed into the 3.8 kernel. Most users will not care, but, for those who obsess about latency and jitter, it should be a welcome addition.
