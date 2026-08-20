---
title: Improving scheduler latency
url: https://lwn.net/Articles/404993/
date: "September 14, 2010"
category: "Latency; Scheduler-Completely fair scheduler; Scheduler-Latency"
author: "By Jonathan Corbet September 14, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 14, 2010

The level of interactive response provided by the kernel's CPU scheduler is the subject of endless discussion and tweaking. It is one of those problems which, seemingly, can never be fully solved to everybody's satisfaction. Some recent discussions on the topic have shown, though, that low-hanging fruit can remain after all these years; it's just a matter of drawing attention to the right place. 

The CFS scheduler divides time into periods, during which each process is expected to run once. The length of the period should thus determine the maximum amount of time that any given process can expect to have to wait to be able to run - the maximum latency. That length, by default, is 6ms. If there are two processes running, those 6ms will be divided up something like this: 

> ![\[Scheduler time slices\]](https://static.lwn.net/images/ns/kernel/sched-granularity1.png)

This assumes that both processes are completely CPU-bound, have the same priority, and that nothing else perturbs the situation, naturally. If a third ideal CPU-bound process shows up, that same period is divided into smaller pieces: 

> ![\[Scheduler time slices\]](https://static.lwn.net/images/ns/kernel/sched-granularity2.png)

This process of dividing the scheduler period cannot continue forever, though. Every context switch has its cost in terms of operating system overhead and cache behavior; switching too often will have a measurable effect on the total throughput of the system. The current scheduler, by default, draws the line at 2ms; if the average time slice threatens to go below that length, the period will be extended instead. So if one more cranker process shows up, the result will be: 

> ![\[Scheduler time slices\]](https://static.lwn.net/images/ns/kernel/sched-granularity3.png)

In other words, once the load gets high enough, the kernel will start to sacrifice latency in order to keep throughput up. In situations where the load is quite high (kernel builds with a lot of parallel processes are often mentioned), latencies can reach a point where users start to get truly irritable. Mathieu Desnoyers decided he could improve the situation with [this patch](<http://lwn.net/Articles/404575/>), which attempted to shrink the minimum possible time slice until there were more than eight running processes; in this way, he hoped to improve latencies on more heavily-loaded systems. 

Mathieu's patch included some test results showing that the maximum latencies had been cut roughly in half. Even so, Peter Zijlstra [dismissed the patch](<https://lwn.net/Articles/405017/>), saying ""Not at all charmed, this look like random changes without conceptual integrity."" That, in turn, earned [a mild rebuke from Linus](<https://lwn.net/Articles/405018/>), who felt that the kernel's latency performance was not as good as it could be. After that, the discussion went on for a while, leading to the interesting conclusion that everybody was partly right. 

Mathieu's patch was based on a slightly flawed understanding of how the scheduler period was calculated, so it didn't do quite what he was expecting. Rejecting the patch was, thus, the correct thing for the scheduler maintainers to do. The patch _did_ improve latencies, though. It turns out that the change that actually mattered was reducing the length of the minimum time slice from 2ms to 750µs. That allows the scheduler to keep the same period with up to eight processes, and reduces the expansion of the period thereafter. The result is better latency measurements and, [it seems](<https://lwn.net/Articles/405025/>), a nicer interactive feel. A patch making just the minimum time slice change was fast-tracked into the mainline and will be present in 2.6.36-rc5. Interestingly, despite the concerns that a shorter time slice would affect throughput, there has not been a whole lot of throughput benchmarking done on this patch so far. 

Things don't stop there, though. One of Mathieu's tests uses the `SIGEV_THREAD` flag to `timer_create()`, causing the creation of a new thread for each event. That new thread, it seems, takes a long time to find its way into the CPU. The culprit here seems to be in the code which tries to balance CPU access between a newly forked process and its parent - a place which has often proved problematic in the past. Mike Galbraith [pointed out](<https://lwn.net/Articles/405029/>) that the `START_DEBIT` scheduler feature - which serves to defer a new task's first execution into the next period - has an unpleasant effect on latency. Turning that feature off improves things considerably, but with costs felt elsewhere in the system; in particular, it allows fork-heavy loads to unfavorably impact other processes. 

Mathieu posted [a patch](<https://lwn.net/Articles/405046/>) adding a new feature called `START_NICE`; if it is enabled, both processes returning from a `fork()` will have their priority reduced for one scheduler period. With that penalty, both processes can be allowed to run in the current period; their effect on the rest of the system will be reduced. The associated benchmark numbers show a significant improvement from this change. 

Meanwhile, Peter went away for a bit and came back with [a rather more complex patch](<https://lwn.net/Articles/405033/>) demonstrating a different approach. With this patch, new tasks are still put at the end of the queue to ensure that they don't deprive existing processes of their current time slices. But, if the new `DEADLINE` feature is turned on, each new task also gets a deadline set to one scheduler period in the future. Should that deadline pass without that process being scheduled, it will be run immediately. That should put a cap on the maximum latency that new threads will see. 

This patch is large and complex, though, and Peter warns that his testing stopped once the code compiled. So this one is not something to expect for 2.6.36; if it survives benchmarking, though, we might see it become ready for the next development cycle.
