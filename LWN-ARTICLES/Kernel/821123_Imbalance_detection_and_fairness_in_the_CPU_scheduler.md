---
title: Imbalance detection and fairness in the CPU scheduler
url: https://lwn.net/Articles/821123/
date: "May 22, 2020"
category: Scheduler
author: "By Jonathan Corbet May 22, 2020 OSPM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 22, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

The kernel's CPU scheduler is good at distributing tasks across a multiprocessor system, but does it do so fairly? If some tasks get a lot more CPU time than others, the result is likely to be unhappy users. Vincent Guittot ran a session at the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM) looking into this issue, with a focus on detecting load imbalances between CPUs and what to do with a workload that cannot be balanced. 

#### Imbalance detection

In the 5.7 kernel, he began, the `runnable_load_avg` signal [has been removed](<https://git.kernel.org/linus/0dacee1bfa70>) in favor of [`runnable_avg`](<https://git.kernel.org/linus/9f68395333ad>), which is the sum of the "runnable" time of every scheduler entity (either an individual task or a control group containing tasks) in a run queue. The runnable time is defined as the time a task actually spends running, but also the time it spends _waiting_ to run. This change addresses a problem that had been seen in capacity tracking when a task is migrated from one CPU to another. 

Specifically, moving a task off of a CPU moves that task's utilization with it, causing that CPU to appear to suddenly have a lot of spare capacity. But if other tasks on the CPU were spending a lot of time waiting to run, that capacity doesn't really exist; the utilization of those tasks was artificially reduced by the fact that they couldn't run as much as they needed to. Including the waiting time prevents that false capacity from appearing when one task moves away, giving the remaining tasks time to expand their actual utilization. The calculation of when a CPU (or set of CPUs) is overloaded now looks at `runnable_avg`, which must exceed the CPU capacity by a threshold before the scheduler will try to move tasks away. 

NUMA balancing is still not using this metric, though, so there is currently a mismatch between normal load balancing and NUMA balancing. That can lead to conflicting decisions at times. It might make sense to change the balancing at the NUMA level, but NUMA nodes can contain a lot of CPUs, and he worries about the impact of summing that many `runnable_avg` values. He has not started working on this problem, but it's at the top of his list. 

Peter Zijlstra noted that developers are still "chasing the fallout" from the changes that have been made so far. Guittot acknowledged that, saying he's not sure if the NUMA issues play into that or not. 

Another issue relates to the threshold used with `runnable_avg`; it is currently a fixed value. But `runnable_avg` is dependent on the number of runnable tasks, since more tasks will lead to more waiting time. That makes it easier to cross the threshold as the number of tasks increases. 

He presented an example to show how the calculations can vary. Imagine _N_ tasks, all with the same utilization. If they all wake up simultaneously and land in the run queue together, the `runnable_avg` value that results will be proportional to _N_ 2. If, instead, each wakes up just as the previous one is completing the work, `runnable_avg` will be directly proportional to _N_. As _N_ grows, the difference between the two scenarios will become large. 

To fix this, he is playing with scaling the threshold by the number of running tasks. That delays the crossing of the threshold and subsequent determination that the CPUs are overloaded. Benchmarking is ongoing, with no significant results to report so far. He's still looking for a benchmark that demonstrates the problem in a useful way. 

#### Fairness with difficult workloads

Guittot then moved on to a fairness problem: how do you balance a case that simply cannot be balanced? Sometimes the granularity of the load on the system just doesn't match the CPU topology. Three tasks running on two CPUs is one example of such a situation. If two tasks are kept on one CPU, they will get half of the running time that the third task gets, which is unfair. This problem only comes about when the system is overloaded. 

Going to a more complex example, he described nine tasks running on an eight-CPU system. This load cannot be balanced, but it should be fair. He ran some tests using the [`rt-app`](<https://github.com/scheduler-tools/rt-app>) benchmark, comparing the amount of work each task was able to complete. The average unfairness he found was about 20%, with one test reaching 40%. Given that the unfairness cannot go above 50%, that is a pretty bad result. 

There are a couple of rules that control when the scheduler will try to move a task to balance the system. The first is that it will look for a task whose utilization is less than twice the calculated imbalance value. [![\[Vincent Guittot\]](https://static.lwn.net/images/conf/2020/ospm/VincentGuittot-sm.png)](<https://lwn.net/Articles/821142/>) In the scenario described here, this rule will almost never find a task to move, causing load balancing to fail. So the second rule kicks in: it moves a task when the number of load-balancing failures exceeds a threshold. At that point, the scheduler is rather less selective when it comes to picking a task. That leads to unfair scheduling. 

Looking at the problem, he found that some CPUs never manage to pull tasks from others; that causes the tasks that are running on those CPUs to get more than their fair amount of time. This seems to be a result of the fact that load balancing happens nearly simultaneously on all CPUs. This also happens at the CPU group level; load balancing at that level also happens at about the same time. But the balancing running within the group will run more quickly, since it has fewer CPUs to consider. That leads to tasks moving around within a group, but rarely between them. 

Another problem is that the same task is often chosen to migrate; it will get less CPU time as a result. There is an unexpected synchronous pattern between the scheduling period and the load balancer that causes the same task to often be waiting to execute when balancing happens. There is a simple fix for both problems: tweak the load-balancing intervals at the various levels so that they don't run simultaneously and don't line up with the scheduling period. 

Another fix is to reduce the active load balancing that happens when normal load-balancing attempts fail. Active load balancing can move tasks that should not necessarily be moved, so it should only be done when it's clear that it makes sense. 

He has also been looking at the `min_vruntime` value on each CPU; this value can be seen as a proxy for how much the least-scheduled task on that CPU has been able to run. If `min_vruntime` is not increasing equally across CPUs, that is a sign of unfair scheduling. This approach does not scale well, since `min_vruntime` only applies to CPU-level run queues rather than tasks or group-level queues. Still, by taking a cue from `min_vruntime`, he was able to reduce the average unfairness on the `rt-app` test to about 15% — better, but not a complete solution. The maximum unfairness fell to 18%, which is a significant improvement. 

So he decided to try a different metric: the ratio of the load average and the utilization average. That gives a good fairness metric, but is not ideal either. There is a big mismatch between the period over which the utilization average is calculated and the load-balancing period; the utilization average is also capped at the max capacity of the CPU. So instead he is looking at "`sched_avg`", the sum of the average utilization of all the run queues. This helps reduce the cases where load balancing bounces tasks quickly between groups. 

This change reduced average unfairness to 12% with a maximum of 16%. But the "always moving the same task" problem is not fully solved though. This could be mitigated by considering each task's utilization average before moving it; a task that has been discriminated against recently will have lower utilization and should not be moved again soon. At this point, he said, fairness appears to be limited by the `imbalance_pct` threshold which keeps load balancing from happening when the imbalance appears to be too small; this is something to look at next. 

#### Questions and comments

After Guittot concluded, Zijlstra said that he had a number of remarks, but that he would save them for email. The alternative, he said, would be to confuse everybody, including himself. There is another possibility that he thinks might be interesting. 

Qais Youssef asked if the fairness issue was specific to long-running tasks. The periods where contention happens might be small, so might not appear with short-running tasks. That suggests that moving tasks around should not happen right away. Guittot agreed that the problem is easier to see with long-running tasks. 

Zijlstra said he has seen fairness problems in high-performance computing workloads, where it is common to spawn a whole set of jobs, then wait for them all to complete. People running these workloads would like the jobs to complete at about the same time; they hate scheduling jitter. If one CPU is running some other, random task (an SSH server daemon, for example) that will slow its job over time. Users in this scenario would like to see these tasks spread across CPUs to maximize the fairness and increase throughput. Making this problem visible, he said, would require introducing some interference when running benchmarks. 

Valentin Schneider noted that this discussion related to symmetric multiprocessing systems. But what about a big.LITTLE system? If you have a machine with four large CPUs and four small ones running eight CPU-hog tasks, four of those tasks will be stuck on the little CPUs. Should the scheduler rotate tasks around to increase fairness? That is a hard one, Guittot said, because there will be no perceived imbalance in the system. Youssef said that a "race to idle" approach might work better than complete fairness in such situations; the right solution is not always entirely clear. 

At that point, the questions were done and the session came to a close.
