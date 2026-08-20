---
title: "The many faces of \"latency nice\""
url: https://lwn.net/Articles/820659/
date: "May 18, 2020"
category: "Latency; Scheduler-Latency"
author: "By Jonathan Corbet May 18, 2020 OSPM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 18, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

A task's "nice" value describes its priority within the completely fair scheduler; its semantics have roots in ancient Unix tradition. Last August, a ["latency nice" parameter](<https://lwn.net/ml/linux-kernel/20190830174944.21741-1-subhra.mazumdar@oracle.com/>) was proposed to provide similar control over a task's response-time requirements. At the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM), Parth Shah, Chris Hyser, and Dietmar Eggemann ran a discussion about the latency nice proposal; it seems that everybody agrees that it would be a useful feature to have, but there is a wide variety of opinions about what it should actually do. 

#### A different kind of nice

Shah started by describing the latency nice value as a per-task attribute that behaves much like the normal nice value. It gives the scheduler a hint about what the task's latency needs are. It can be tweaked via the `[sched_setattr()](<http://man7.org/linux/man-pages/man2/sched_getattr.2.html>)` system call, though there is some desire to switch to a control-group interface. Its values vary between -20 and 19 (as with nice), with -20 indicating a high degree of latency sensitivity and 19 indicating a complete indifference to latency. The default value is zero. 

The first question he raised had to do with privilege: should an unprivileged process be able to decrease its latency nice value? Ordinary nice does not allow that, of course; processes must have the `CAP_SYS_NICE` capability to reduce their nice values. The advantage of establishing a similar rule for latency nice is that it might block potential denial-of-service problems, but at the cost of preventing ordinary users from taking advantage of this feature. 

Whether this knob should be privileged depends on what it actually does, which had not yet been discussed. The initial effect of this feature is to control how hard the scheduler will look for an idle core to place a task on when it wakes up. This search takes time (thus increasing latency); an idle core may also have to be roused out of a sleep state, increasing latency further. Dhaval Giani pointed out a use case that Oracle cares about, where some latency-sensitive tasks will typically run for very short periods — less than the time spent searching for an idle core sometimes. That search can be avoided by setting a low latency nice value. 

Giani also mentioned a use case from Facebook, which is more interested in getting longer-running tasks up to full speed quickly; Facebook still wants low latency, but is better served by finding an idle core that will be able to get a significant amount of work done quickly. IBM, meanwhile is hoping to use this knob to influence the scheduler to avoid placing tasks on a CPU that is currently running [![\[Latency nice
session\]](https://static.lwn.net/images/conf/2020/ospm/latency-nice-sm.png)](<https://lwn.net/Articles/820743/>) latency-sensitive tasks. The discussion on use cases was cut off at this point, though, with a promise to revisit it later. 

Returning to privilege, Qais Youssef suggested keeping the ability to reduce latency nice values as a privileged operation for now, especially given that this knob could gain new meanings in the future. Shah said that there do not appear to be any denial-of-service issues with the implementation for the current use cases. 

Eggemann wondered about the range of values for this knob; there is a wish to bias latency in both directions, but it's not clear what the actual effects of a positive latency nice value would be. Patrick Bellasi suggested that the time before one task could preempt another could be scaled by the latency nice value. Vincent Guittot said that, with ordinary nice, each increment makes about a 10% difference in the amount of CPU time the process may use. With latency nice, he said, the values of -19, zero, and +20 make sense, but he couldn't say what the values in between would mean. Hyser said that, for negative values, there could be a fairly direct effect on the number of CPUs that will be searched before placing a task. Shah suggested that positive values could allow task placement anywhere in the system, even to CPUs that do not share low-level memory cache, which is something the scheduler normally tries hard to avoid. 

Eggemann then expressed a sentiment that would be heard a few times in the session: latency nice is trying to control too many functionalities with a single knob. Bellasi suggested that the use cases could be hammered out during review of the patch and asked whether there were any real use cases with contradictory semantics. Giani mentioned the Oracle and Facebook cases mentioned above. 

#### Control groups

Eggemann took over the presentation at this point to talk about what the Android developers would like to see. Android currently uses a control-group interface that includes a "prefer idle" attribute; setting that will bias CPU selection toward an idle CPU. The real effect of this setting, though, is to short out the energy-aware scheduling logic, which brings a certain amount of latency of its own. Thus, in this context, searching for an idle CPU is something that is preferable to do for latency-sensitive tasks — just the opposite of the situation described above. 

His real purpose, though, was to discuss a potential control-group-based interface for latency nice. Control groups are a mechanism to organize processes and distribute resources, which is what is needed here. With the CPU controller, there are three ways in which CPU resources are controlled. The "weight" value gives a relative priority to the group, while the "max" value limits the maximum CPU time available and the "min" value ensures that a minimal amount of CPU time will be granted. Utilization clamping is also handled here. 

How could the latency nice value be managed in this setting? The resource controlled would still have to be CPU cycles, he said. But the association between latency requirements and CPU cycles is not as clear as it is with the parameters described above. He is not sure what sort of semantics would be acceptable to the control-group maintainer. Bellasi suggested a clamping model, where each group would have values indicating the minimum and maximum latency nice values a task in that group could request. Guittot pointed out, though, that changes to latency nice values would have to be propagated up to the root of the control-group hierarchy. The discussion wandered around this point for a while before bogging down in just how latency nice would work 

Eggemann eventually suggested moving on, saying that perhaps the use cases should have been discussed from the outset. The control-group interface is only really important to Android, he said, so perhaps it would be better to figure out what the per-task attribute implementation would actually be doing. 

#### Use cases at last

Hyser took over at this point to talk about use cases; he reiterated that the original purpose of the patch set was to skip the idle-CPU search for latency-sensitive tasks. This resulted in a 1% increase in a transaction-processing benchmark. Many workloads have critical processes that do not run for long but need to run immediately when the time comes. The latency nice change can make it possible for many of these workloads to avoid the need to use the realtime patches. 

He put up some plots showing that latency nice does result in better latencies; the effect is more pronounced on systems with more cores. 

> ![\[Latency plot\]](https://static.lwn.net/images/conf/2020/ospm/lnice.png)

He suggested that negative values should be interpreted as the number of cores to search; a value of -20 means search no cores at all, -19 would search one core, etc. But should this value be scaled by the number of CPUs in the system? It's still not clear how it should be interpreted. He suggested that latency nice looks a lot like a Boolean value in real-world use; either other cores are searched to place a task or not. 

Giani said that the effect of changing a task's nice value is well understood; the effect of changing latency nice is rather less so. Hyser suggested that it could be seen as adjusting the size of the scheduling domain for latency-sensitive tasks. But scheduling domains are hardware dependent, making it hard to come up with a hardware-independent description of the semantics of latency nice. The -20 value, which searches zero cores, is not dependent on hardware at least, Hyser said. He concluded by saying that a value of -1 could mean that the CPU search would happen, but energy-aware scheduling would be disabled. 

Giani said that latency nice appears to be trying to do a bunch of things and wondered if it makes sense to control it all with a single interface; Peter Zijlstra responded that those things do all affect latency, at least. Rafael Wysocki said that a single integer value is not enough to express everything that is needed here. Zijlstra said that the session really should have started with the use cases, then looked at tunables to suit those cases. 

Shah discussed the task-packing use case. In particular, on systems with Intel's "turbo" mode, packing tasks onto a small number of cores [can save enough resources](<https://lwn.net/Articles/792471/>) to allow others to go into turbo mode. He suggested that tasks marked with a latency nice value greater than 15 could be packed this way, as long as they don't push the utilization of the target core above a threshold value. Doing so led to a 14% performance benefit on a workload he tested. 

Another use case involves restricting the sleep states that a CPU can go into. The [pm_qos](<https://lwn.net/Articles/386139/>) mechanism can do that now, but it is a system-wide parameter with no per-task control, so it does not work as well as one would like on larger systems; it has no notion of where the latency-sensitive task will run. He suggested implementing a per-CPU counter indicating how many latency-sensitive tasks are present; if a CPU is running such tasks, the sleep states it could go into would be restricted. 

Wysocki responded that this isn't a realistic scenario. It could become confused if the task is migrated, for example; he said that latency nice is not a good interface for this case. There is no way to map a latency nice value and the set of permissible exit latencies for the CPU. Bundling semantics in this way is not going to work, he said. Bellasi said that such an interface would require users to determine their latency nice values through experimentation on a specific platform. 

Shah persisted, though, saying that it can be beneficial to keep CPUs with latency-sensitive tasks from going idle. Scheduler benchmark runs showed a significant latency reduction with these semantics while maintaining similar power consumption. A pgbench run also showed big improvements in latency, but at a cost (sometimes large) in power consumption. 

Youssef said that the interface to all of this is the sticking point. Thomas Gleixner agreed, saying that the -20..19 range "requires a crystal ball" to use properly. Zijlstra repeated his call to enumerate the use cases before getting into the interface details. Giani repeated that the interface does not look correct now, and agreed that a more comprehensive look at the use cases was needed. Things were being done backwards currently, he said. 

Eggemann concluded by saying that the group needed to collect use cases and "take them all seriously". While the discussion continued to circle around these points for a while, it was, for all practical purposes, done. 

[See [the slides from this session [PDF]](</images/conf/2020/ospm/latency-nice-slides.pdf>) for more plots and other details.]
