---
title: Soft CPU affinity
url: https://lwn.net/Articles/792502/
date: "July 4, 2019"
category: "Scheduler-CPU affinity"
author: "By Jonathan Corbet July 4, 2019"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 4, 2019

On NUMA systems with a lot of CPUs, it is common to assign parts of the workload to different subsets of the available processors. This partitioning can improve performance while reducing the ability of jobs to interfere with each other. The partitioning mechanisms available on current kernels might just do too good a job in some situations, though, leaving some CPUs idle while others are overutilized. The [soft affinity patch set](<https://lwn.net/ml/linux-kernel/20190626224718.21973-1-subhra.mazumdar@oracle.com/>) from Subhra Mazumdar is an attempt to improve performance by making that partitioning more porous. 

In current kernels, a process can be restricted to a specific set of CPUs with either the [`sched_setaffinity()`](<http://man7.org/linux/man-pages/man2/sched_setaffinity.2.html>) system call or the [cpuset](<http://man7.org/linux/man-pages/man7/cpuset.7.html>) mechanism. Either way, any process so restricted will only be able to run on the specified CPUs regardless of the state of the system as a whole. Even if the other CPUs in the system are idle, they will be unavailable to any process that has been restricted not to run on them. That is normally the behavior that is wanted; a system administrator who has partitioned a system in this way probably has some other use in mind for those CPUs. 

But what if the administrator would rather relax the partitioning in cases where the fenced-off CPUs are idle and going to waste? The only alternative currently is to not partition the system at all and let processes roam across all CPUs. One problem with that approach, beyond losing the isolation between jobs, is that NUMA locality can be lost, resulting in reduced performance even with more CPUs available. In theory the [AutoNUMA](<https://lwn.net/Articles/488709/>) balancing code in the kernel should address that problem by migrating processes and their memory to the same node, but Mazumdar notes that it doesn't seem to work properly when memory is spread out across the system. Its reaction time is also said to be too slow, and the cost of the page scanning required is high. 

So Mazumdar has taken a different approach with a patch set that tries to resolve the issue by creating a concept of "soft affinity". It starts by adding a new system call: 

```
int sched_setaffinity2(pid_t pid, size_t cpusetsize, cpu_set_t *mask,
    			   int flags);
```

The first three arguments mirror `sched_setaffinity()`: they identify the process to be modified and provide a mask of CPUs on which that process can run. The `flags` argument is new, though. If it is set to `SCHED_HARD_AFFINITY`, then this call will behave just like `sched_setaffinity()`, absolutely restricting the processes to the CPUs in the given `mask`. `SCHED_SOFT_AFFINITY`, instead, sets a new "soft affinity" mask (which must be a subset of the hard mask) and thereby requests the new behavior. 

Said behavior is to treat the soft-affinity CPU mask the same as old-style "hard" affinity most of the time: the process will run only on the CPUs listed in that mask. If, however, those CPUs are busy and other CPUs in the process's hard-affinity mask are close to idle, the process will be allowed to run on the idle CPUs as well. That allows the workload to spread out across the system, but only when CPUs are underutilized. 

In other words, this patch creates two levels of CPU affinity masks, where in current kernels there is only one. Both masks default to containing all CPUs in the system (as the hard mask does in current kernels). The behavior of the hard mask is unchanged, but the new soft mask can be used to further restrict processes to a smaller group of CPUs; that further restriction can be relaxed by the kernel at times when CPUs found only in the hard mask are idle. 

The decision on whether to allow a constrained process to "break out" of its soft-affinity mask is based on two new sysctl knobs, called `sched_allowed` and `sched_preferred`. If the ratio of `sched_allowed` to `sched_preferred` is greater than the ratio of the CPU utilization of the soft-affinity CPUs to that of another CPU, then that other CPU will be considered for placing a task. The default is to set `sched_allowed` to 100 and `sched_preferred` to one, meaning that a CPU outside of the soft-affinity set must be only 1% as loaded as the CPUs inside the set before a soft-affinity process will be moved there. That ratio is a pretty high bar; the target CPU would have to be idle indeed to pass this test. In sites where this mechanism is used, the administrator would probably want to tune those parameters differently. 

One question not addressed within the patch set is what happens when a process that has been moved out of the soft-affinity CPUs inevitably raises the utilization of the CPU it is moved to. The soft-affinity decision is made whenever a process wakes up, so the expected behavior would seem to be that the process would run on the outside CPU until it sleeps again. If that sleep is relatively short, it seems likely that the process would be moved again on its next wakeup. 

Benchmarks provided with the patch set show performance increases of up to about 7% for some workloads, and regressions in some others. Most of the improvements are relatively small, though, and the data seems noisy. Reviewers might also look closely at Mazumdar's claim that the AutoNUMA balancing does not do a good enough job and, in particular, whether it would be better to improve that code rather than adding a new mechanism and a new set of scheduler tunables. 

So it is not clear that the case for this work has yet been made convincingly, something that would need to happen before this work would be considered for merging. Work along these lines seems destined to continue, though. The pressure to get as much work as possible out of every CPU seems unlikely to decrease, and even a relatively small performance improvement is worth a fair amount when it is replicated across a large data center. Soft affinity may or may not be an answer to this problem, but it is indicative of the needs that are driving kernel development for those environments.
