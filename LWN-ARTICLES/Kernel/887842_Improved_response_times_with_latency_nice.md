---
title: Improved response times with latency nice
url: https://lwn.net/Articles/887842/
date: "March 17, 2022"
category: "Latency; Scheduler-Latency"
author: "By Jonathan Corbet March 17, 2022"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 17, 2022

CPU scheduling can be a challenging task; the scheduler must ensure that every process gets a fair share of the available CPU time while, at the same time, respecting CPU affinities, avoiding the migration of processes away from their cached memory contents, and keeping all CPUs in the system busy. Even then, users can become grumpy if specific processes do not get their CPU share quickly; from that comes years of debates over desktop responsiveness, for example. The [latency-nice priority proposal](<https://lwn.net/ml/linux-kernel/20220311161406.23497-1-vincent.guittot@linaro.org/>) recently resurrected by Vincent Guittot aims to provide a new tool to help latency-sensitive applications get their CPU time more quickly. 

Over the years, numerous approaches have been used to try to improve the response time of important processes. The traditional Unix "nice" value can be used to raise a process's priority, for example. That can work, but a process's niceness does not directly translate into latency; it controls how much of the available CPU time the process can consume, but not when the process can actually run. Using the realtime priorities _will_ cause the scheduler to run a process quickly, especially if realtime preemption is enabled, but a process running at that priority can also take over the system. 

The latency-nice concept is a different approach that tries to address those problems; it applies to the completely fair scheduler used for most processes, so no realtime priorities are needed. It adds a second nice value which, mirroring the existing nice value, is a number between -20 and 19\. The lower the number, the higher the priority, so the highest-priority latency-nice value is -20. As with traditional nice values, any process can increase its latency-nice setting, but lowering it requires the `CAP_SYS_NICE` capability. 

The traditional nice value works by regulating how much CPU time a process may consume relative to others on the system; processes with a lower nice value get more CPU time. Changing the latency-nice value, instead, does not change the amount of CPU time a process may consume. It does, however, make a difference in _when_ that time will be made available. Processes with lower latency-nice values are deemed to be more latency-sensitive, and thus should not have to wait as long before being able to use the CPU time that is available to them. 

With that model, the implementation of latency nice is relatively straightforward. Whenever a blocked process wakes, the scheduler must decide whether to run it immediately or to put it into a run queue and make it wait for a CPU. A number of factors go into that decision now; the latency-nice mechanism adds another. If the new process has a higher latency-nice priority than the process that is running in a CPU, and that new process has available CPU time in its current slice, then the new process can preempt the running process. The new process does not get any more CPU time than before, but it has the right to obtain the CPU more quickly when it has time available. 

Similarly, a process with a higher latency-nice value (and thus, a lower priority) will not preempt other running processes. It will thus tend to get its entire time allotment toward the end of the slice, once the higher-priority processes have used their time. This process, too, will get all of the time that it is entitled to, but it will not block others and will, because it does not preempt others, cause fewer context switches in general. 

Traditional nice values are set with the [`nice()` system call](<https://man7.org/linux/man-pages/man2/nice.2.html>). Latency nice, instead, is controlled with [`sched_setattr()`](<https://man7.org/linux/man-pages/man2/sched_setattr.2.html>). A new field (`latency_nice`) has been added to the `sched_attr` structure passed to that system call, and the `SCHED_FLAG_LATENCY_NICE` flag is provided to indicate that a new latency-nice value is being requested. Latency nice can also be managed using the scheduler control-group controller; a new knob (`latency`) has been provided for that purpose. 

[This patch](<https://lwn.net/ml/linux-kernel/20220311161406.23497-6-vincent.guittot@linaro.org/>) in the series includes some benchmark results showing how latency nice works. Running the `hackbench` benchmark with a high latency-nice value yields better performance due to the lower number of preemptions that take place. Throwing in a `cyclictest` run, at a low latency-nice value, demonstrates greatly reduced latency results for that test. Overall, it would seem that the patch set works as intended. 

Previously, this work had been developed by Parth Shah; the [fifth revision of the patch set](<https://lwn.net/ml/linux-kernel/20200228090755.22829-1-parth@linux.ibm.com/>) was posted in February 2020. The work had acquired some Reviewed-by tags by that point, but it stalled thereafter. Interestingly, it had gotten as far as adding the infrastructure to manage the latency-nice value, but had not actually implemented any new semantics in the scheduler. At that time, there were a few ideas circulating on how the system might respond to the latency-nice settings and [a discussion on latency nice](<https://lwn.net/Articles/820659/>) was held at the OSPM 2020 gathering, but no seeming consensus on the right approach emerged. 

Two years later, Guittot has dusted this work off and added the wakeup implementation described above. As of this writing, there have been few comments on this work. Improving response times for important processes has been on many developers' wishlists for a long time, though. If further testing shows that the latency-nice mechanism represents progress in that direction, then this new push may well be the one that gets this work into the mainline kernel.
