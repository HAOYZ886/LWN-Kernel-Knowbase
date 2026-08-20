---
title: SMPnice
url: https://lwn.net/Articles/186438/
date: "June 6, 2006"
category: Scheduler
author: 
---

A great deal of work has gone into making the Linux scheduler work well on multiprocessor systems. Whenever it appears to make sense, the scheduler will shift processes from one CPU to another in order to keep all CPUs equally busy (in an approximate sense), but, since moving a process is expensive, the scheduler tries to avoid unnecessary moves. SMP performance was problematic on early 2.6 releases, but it has been reasonably solid for the last couple of years. 

There is one situation, however, where the current scheduler does not work as well as one would like. Imagine a simple system with two processors. If two CPU-bound processes, each running at normal priority, are started on this system, the scheduler will eventually run one process on each CPU. If two niced (low-priority) processes (also CPU-bound) are then started, one would normally expect the scheduler to ensure that those processes get less CPU time than the normal-priority processes. 

If the processes are distributed such that one normal-priority and one low-priority process end up on each CPU, that expectation will be met; the low-priority processes will get a relatively small amount of CPU time. It is just as likely, however, that both normal-priority processes will end up on the same CPU, with the two low-priority processes on the other. In this case, the two normal-priority processes will be contending for the same CPU, while the low-priority processes fight for the other. As a result, the low-priority processes will get as much CPU time as the others, their reduced priority notwithstanding. That is almost certainly not what the user had in mind when the process priorities were set. 

The problem is that the scheduler looks only at the length of the run queue on each CPU, without taking priorities into account. So, in either case above, the CPUs appear to be equally busy, and no redistribution of processes will occur. To fix this problem, the load balancing code must be made to understand that not all running processes are created equal. 

A solution can be found in the "smpnice" patch set, implemented by Peter Williams with input from a number of other developers. The smpnice code changes the load balancer so that it does not just look at run queue lengths. Instead, each process is assigned a "load weight," which is derived from its priority. When load balancing decisions are made, the scheduler compares total load weights rather than the length of the run queues. If a load weight imbalance is detected, the scheduler will move a process to bring things back into line. If the imbalance is large, high-priority processes will be moved; when the imbalance is small, however, a low-priority process will be moved instead. 

The basic idea makes sense, but this set of patches has been a long time in development. The scheduling code is full of subtle heuristics which are easily upset. So early versions of the smpnice patches caused benchmark regressions and ran into a number of difficulties. For example, a processor running a very high-priority process will tend to appear to be the most heavily loaded, with the result that load balancing no longer occurs between other processors on the system. This problem was fixed by ignoring processors which have no processes which can be moved. Some load balancing heuristics which would move high-priority processes were broken, resulting in suboptimal scheduling decisions; now, if a process would have the highest priority on the new CPU, it is considered first for moving. Various stability problems, where processes would oscillate between processors, have also been ironed out. 

With all of these fixes applied, the smpnice code appears to be stabilizing, with the result that it might just make it into the 2.6.18 kernel. That should improve life for people running multiple-priority workloads on SMP systems.
