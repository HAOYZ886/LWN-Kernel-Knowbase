---
title: Tracking tbench troubles
url: https://lwn.net/Articles/304845/
date: "October 29, 2008"
category: "Benchmarking; Scheduler-Testing and benchmarking"
author: "By Jonathan Corbet October 29, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 29, 2008

Kernel developers tend to have a mixed view of benchmarks. A benchmarking tool can do an effective job of quantifying specific aspects of system performance. But benchmarks are not real workloads; optimizing for a benchmark can often distort a system in ways which are detrimental to real applications. Since kernel hackers do not always see benchmark optimization as their top priority, they can sometimes assign a lower priority to benchmark regressions as well. But, sometimes, benchmark problems indicate a real problem in the kernel. 

The tbench benchmark is meant to measure networking performance; it consists of a collection of processes quickly making lots of small requests from a server process. Since the requests are small, there is not much time spent actually moving data; it's all a matter of shifting small packets around - and scheduling between the processes. Back in August, Christoph Lameter [reported](<http://marc.info/?l=linux-kernel&m=121847986119495&w=4>) that tbench performance in the mainline kernel had been declining for some time. His system was able to move 3208 MB/sec with a 2.6.22 kernel, but only 2571 MB/sec with a 2.6.27-rc kernel. Each of the releases in between showed a decline from the one which came before, with 2.6.25 showing an especially big hit. Others were able to reproduce the results, and they engaged in various rounds of speculation on where the problem might be, but it seems that, initially, nobody actually dug into the system to see what was going on. 

At linux.conf.au 2007, Andi Kleen gave a talk describing various types of kernel hackers. One of those was the "Russian mathematician" who, he suspected, was often a room full of talented developers operating under a single name. Evgeniy Polyakov can only have reinforced that view when, in early October, he [tracked down the biggest offending commit](<https://lwn.net/Articles/304847/>) through a process which, he says, involved "just [a] couple of hundreds of compilations." In the process, he put together [a plot of tbench performance](<http://www.ioremap.net/node/37>) which, he says, is suitable for scaring children. Through a massive amount of work, he was able to point the finger at a scheduler patch - not something in the networking stack at all. 

In particular, Evgeniy found that the patch adding high-resolution preemption ticks was the problem. The idea behind this patch was to make time slices more accurate by scheduling preemption at just the right time. It makes sense; once the regular clock tick has been eliminated, there is no reason not to arrange for preemption to happen when the scheduling algorithm says it should. Unfortunately, it seems that this change also adds sufficient overhead to slow down tbench performance considerably; when Evgeniy backed it out, his performance went from 373 MB/sec to 455 MB/sec. That would seem to be a pretty clear indication that something is amiss with high-resolution preemption ticks. 

At this point, the public discussion went quiet, though it appears that a number of developers were working on it off-list. David Miller eventually tracked down the worst of the trouble to the wakeup code, something he was [rather vocally unhappy](<https://lwn.net/Articles/304873/>) about having had to do. Eventually a patch was merged (for 2.6.28-rc2) disabling the high-resolution preemption tick feature. Since the discussion is private, it's not quite clear why this change took as long as it did. But there's a couple of plausible reasons. One is that this particular feature is disabled by default anyway, so most users will not encounter the performance problem it creates. 

But there is also the question of weighing the benchmark result against the effects on other, "real" workloads. Ingo Molnar [said](<https://lwn.net/Articles/304875/>): 

But it's a difficult call with no silver bullets. On one hand we have folks putting more and more stuff into the context-switching hotpath on the (mostly valid) point that the scheduler is a slowpath compared to most other things. On the other hand we've got folks doing high-context-switch ratio benchmarks and complaining about the overhead whenever something goes in that improves the quality of scheduling of a workload that does not context-switch as massively as tbench. It's a difficult balance and we cannot satisfy both camps. 

So, by this view, performance on scheduler-intensive benchmarks must be weighed against the wider value of other scheduler enhancements. David Miller [has a different view](<https://lwn.net/Articles/304876/>) of the situation, though: 

If we now think it's ok that picking which task to run is more expensive than writing 64 bytes over a TCP socket and then blocking on a read, I'd like to stop using Linux. :-) That's "real work" and if the scheduler is more expensive than "real work" we lose. 

In David's view, scheduler performance has been getting consistently worse since the switch to the completely fair scheduler in 2.6.23. He would like to see some energy put into recovering some of the performance of the pre-CFS scheduler; in particular, he thinks that Ingo and company should work to fix (what he sees as) a regression that they caused. 

For the time being, the worst performance regression has been "fixed" by disabling the high-resolution preemption tick feature; Ingo says that the feature will not come back until it can be supported without slowing things down. But the scheduler seems to have gotten slower in a number of other ways as well. Your editor will make a prediction here: now that the issue has been called out in such clear terms, somebody will find the time to fix these problems to the point that the CFS scheduler will be faster than the O(1) scheduler which preceded it. 

Beyond that, there are [suggestions](<https://lwn.net/Articles/305092/>) that the scheduler cannot take the blame for all of the observed regressions in tbench results. So developers will have to look at the rest of the system to figure out what's going on. The good news is that this is a clear challenge with an objective way to measure success. Once a problem reaches that level of clarity, it's usually just a matter of some hacking.
