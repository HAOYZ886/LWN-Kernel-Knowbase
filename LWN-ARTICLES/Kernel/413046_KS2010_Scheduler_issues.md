---
title: "KS2010: Scheduler issues"
url: https://lwn.net/Articles/413046/
date: "November 3, 2010"
category: Scheduler
author: "By Jonathan Corbet November 3, 2010 2010 Kernel Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 3, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

Google, as everybody knows, does not have very many computers sitting around, so it is not surprising that the company tries to cram as much work as possible onto each available CPU. That desire leads to the mixing of various types of workloads; a single system may be doing low-priority work like video transcoding and also handling high-priority search requests. The group scheduling feature is being used to help make these workloads perform properly. But, as Google's Paul Turner noted, group scheduling is "a fairly immature extension" which has some remaining glitches. 

Among those are the fact that group scheduling bundles both bandwidth (the amount of CPU time allocated to a group) and priority into a single value. There are some real scalability issues with group scheduling; the wakeup path, in particular, is getting costly. Paul complained about a lack of cooperative scheduling APIs. Management of group scheduling is difficult; for the desktop case, automatic tty-based grouping will make life easier, but it won't help on server systems. There is no notion of priority between groups and no upper bound on the bandwidth any given group can consume. There are load balancing problems, especially when networking comes into the picture. And there is no notion of idle or batch scheduling in the group context. 

With regard to load balancing, Paul said that the weight-based balancing tends to hurt CPU utilization. The balancing of groups is "primitive," leading to "herd migrations" which don't help the problem. There is no NUMA awareness in the group scheduler. The scheduler also does not account for the CPU time consumed by interrupt handling, leading to skewed scheduling results. Threaded interrupt handlers were suggested as a possible way of mitigating that last problem. 

Google wants to use `SCHED_IDLE` for low-priority tasks, but it works poorly with the load balancing. Since idle tasks have no weight, the scheduler will not move them over to an idle core. These tasks also get a minimum share of the CPU which, while small, is still too high; it is not possible to isolate those loads entirely from the rest of the system. 

Talking about scalability, Paul called out `tg_shares_up()`, which handles the distribution of CPU bandwidth. It is costly; since it is running across the Google cluster, he said, it may well be the function which is consuming the most CPU time on the planet. Something needs to be done to streamline that part of the system. Wakeup costs are high too; Paul would like to find a way to offload some of that cost to the CPU where the target process is running. That, he says, would spread out the costs and reduce cross-processor lock contention. 

Google has posted some patches which allow the specification of an upper bound for CPU utilization; Paul would like to see that work merged. He would like to see the addition of priorities to group scheduling. Also nice would be a means by which the fairness window could be different for each group. High-priority groups should be given their fair share with relatively small periods; low priority work really only needs its share over longer periods. 

Paul also talked about yet another variant on deadline scheduling called EEVDF. It works with virtual deadlines, so it's not aimed at realtime use. But it enables the sort of scheduling that Google would like, and it mixes very well with the current CFS scheduler. Evidently it provides non-uniform latency periods - implementing the variable windows that Google would like - and has nice idle-scheduling behavior as well. 

Then, there was talk of "cooperative scheduling," which includes a mechanism by which threads can be notified when they are preempted or migrated. The notification mechanism was not clearly described; it sounded like a variant on signals. There is also a desire for a "thread nomination" mechanism by which one thread can pick another to run at any given time. 

There was also some talk of testing, which, Paul said, is hard. One thing that has helped a lot is [linsched](<https://lwn.net/Articles/409680/>), a scheduler simulator which has recently been fixed up and posted by Google. Linsched makes it easy to run tests in a highly repeatable way. 

[Next: Kernel.org update](<https://lwn.net/Articles/413059/>).
