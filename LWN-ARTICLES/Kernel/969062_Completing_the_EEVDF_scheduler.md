---
title: Completing the EEVDF scheduler
url: https://lwn.net/Articles/969062/
date: "April 11, 2024"
category: "Releases-6.12; Scheduler-EEVDF"
author: "By Jonathan Corbet April 11, 2024"
---

By **Jonathan Corbet**  
April 11, 2024

The [Earliest Virtual Deadline First (EEVDF) scheduler](<https://lwn.net/Articles/925371/>) was merged as an option for the 6.6 kernel. It represents a major change to how CPU scheduling is done on Linux systems, but the EEVDF front has been relatively quiet since then. Now, though, scheduler developer Peter Zijlstra has returned from a long absence to post [a patch series](<https://lwn.net/ml/linux-kernel/20240405102754.435410987@infradead.org/>) intended to finish the EEVDF work. Beyond some fixes, this work includes a significant behavioral change and a new feature intended to help latency-sensitive tasks. 

#### A quick EEVDF review

The EEVDF scheduler works to divide the available CPU time equally between all of the runnable tasks in the system (assuming all have the same priority). If four equal-priority tasks are contending for the CPU, each will be given a 25% share. Every task is assigned a virtual run time that represents its allocated share of the CPU; in the four-task example, the virtual run time can be thought of as a clock that runs at 25% of wall-clock speed. As tasks run, the kernel computes the difference between a task's virtual run time and its actual running time; the result is called "lag". A positive lag value means that a task is owed CPU time, while a negative value indicates that a task has received more than its share. 

A task is deemed "eligible" if its lag value is zero or greater; whenever the CPU scheduler must pick a task to run, it chooses from the set of eligible tasks. For each of those tasks, a virtual deadline is computed by adding the time remaining in its time slice to the time it became eligible. The task with the earliest virtual deadline will be the one that is chosen to run. Since a longer time slice will lead to a later virtual deadline, tasks with shorter time slices (which are often latency sensitive) will tend to run first. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

An example might help to visualize how lag works. Imagine three CPU-bound tasks, called A, B, and C, that all start at the same time. Before any of them runs, they will all have a lag of zero: 

> | A| B| C  
> ---|---|---|---  
> Lag:| 0ms | 0ms | 0ms  
  
Since none of the tasks have a negative lag, all are eligible. If the scheduler picks A to run first with a 30ms (to pick a number) time slice, and if A runs until the time slice is exhausted, the lag situation will look like this: 

> | A| B| C  
> ---|---|---|---  
> Lag:| -20ms | 10ms | 10ms  
  
Over those 30ms, each task was entitled to 10ms (one-third of the total) of CPU time. A actually got 30ms, so it accumulated a lag of -20ms; the other two tasks, which got no CPU time at all, ended up with 10ms of lag, reflecting the 10ms of CPU time that they should have received. 

Task A is no longer eligible, so the scheduler will have to pick one of the others next. If B is given (and uses) a 30ms time slice, the situation becomes: 

> | A| B| C  
> ---|---|---|---  
> Lag:| -10ms | -10ms | 20ms  
  
Once again, each task has earned 10ms of lag corresponding to the CPU time it was entitled to, and B burned 30ms by actually running. Now only C is eligible, so the scheduler's next decision is easy. 

One property of the EEVDF scheduler that can be seen in the above tables is that the sum of all the lag values in the system is always zero. 

#### Lag and sleeping

The lag calculation is only relevant for tasks that are runnable; a task that sleeps for a day is not actually missing out on its virtual run time (since it has none), so it does not accumulate a huge lag value. The scheduler does, however, retain a task's current lag value when it goes to sleep, and starts from that value when the task wakes. So, if a task had run beyond its allocation before it sleeps, it will pay the penalty for that later, when it wakes. 

There does come a point, though, where it may not make sense to preserve a task's lag. Should that task that just slept for a day really be penalized for having been allowed to run a bit over its allocation yesterday? It seems clear that, sooner or later, a task's lag should revert back to zero. But when that should happen is not entirely clear. As Zijlstra pointed out in [this patch](<https://lwn.net/ml/linux-kernel/20240405110010.631664251@infradead.org/>) from the series, forgetting lag immediately on sleep would make it possible for tasks to game the system by sleeping briefly at the end of their time slice (when their lag is probably negative), with the result that they get more than their share of CPU time. Simply decaying the lag value over time will not work well either, he concluded, since lag is tied to virtual run time, which passes at a different (and varying) rate. 

The solution is to decay a sleeping task's lag over virtual run time instead. The implementation of this idea in the patch set is somewhat interesting. When a task sleeps, it is normally removed from the run queue so that the scheduler need not consider it. With the new patch, instead, an _ineligible_ process that goes to sleep will be left on the queue, but marked for "deferred dequeue". Since it is ineligible, it will not be chosen to execute, but its lag will increase according to the virtual run time that passes. Once the lag goes positive, the scheduler will notice the task and remove it from the run queue. 

The result of this implementation is that a task that sleeps briefly will not be able to escape a negative lag value, but long-sleeping tasks will eventually have their lag debt forgiven. Interestingly, a positive lag value is, instead, retained indefinitely until the task runs again. 

#### Time-slice control

As noted above, tasks with a shorter time slice will have an earlier virtual deadline, causing them to be selected sooner by the scheduler. But, in current kernels, that implicit priority only takes effect when the scheduler is looking for a new task to run. If a latency-sensitive task with a short time slice wakes up, it may still have to wait for the current task to exhaust its time slice (which might be long) before being able to run. Zijlstra's patch series changes that, though, by allowing one task to preempt another if its virtual deadline is earlier. This change provides more consistent timings for short-time-slice tasks, while perhaps slowing long-running tasks slightly. 

That leaves one open question, though: how does one specify that a given task should be given a short time slice? In current kernels, there is no way for a non-realtime process to tell the kernel what its time slice should be, so this patch series adds that capability. Specifically, a task can use the [`sched_setattr()` system call](<https://man7.org/linux/man-pages/man2/sched_setattr.2.html>), passing the desired slice time (in nanoseconds) in the `sched_runtime` field of the `sched_attr` structure. This field, in current kernels, is only used for [deadline scheduling](<https://lwn.net/Articles/743740/>). With this addition, any task can request shorter time slices, which will cause it to be run sooner and, possibly, more frequently. If, however, the requested time slice is too short, the task will find itself frequently preempted and will run slower overall. 

The allowed range for time slices is 100µs to 100ms. For the curious, Zijlstra has illustrated the results of various time-slice choices as an impressive set of ASCII-art diagrams in the changelog for [this patch](<https://lwn.net/ml/linux-kernel/20240405110010.934104715@infradead.org/>). 

These changes are in a relatively early state and seem likely to require revisions before they can be considered for merging. Among other things, the interaction with control groups has not yet been investigated and may well not work properly. But, once the details have been taken care of, the EEVDF scheduler should be getting to the point where it is ready for wider use.
