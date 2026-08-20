---
title: NoHZ tasks
url: https://lwn.net/Articles/420544/
date: "December 20, 2010"
category: "Dynamic tick; HZ; Read-copy-update"
author: "By Jonathan Corbet December 20, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 20, 2010

Operating system kernels, at their best, should not be noticed by user space at all; in particular, the resource cost of the kernel should be as small as possible. The Linux kernel has been written with that idea in mind, but, for some people, anything is still too much. High-performance computing users want all of the CPU time for themselves, while some latency-sensitive users want their code to never have to wait for the processor. These users have been asking for a way to run processes on at least one CPU with no kernel interference at all - no timer ticks, no interrupts, etc. Thus far, no satisfactory solution has been found; a new patch set by Frederic Weisbecker is not such a solution yet, but it shows another way of attacking the problem. 

The idea behind Frederic's [patch set](<https://lwn.net/Articles/420490/>) is to enable a process to disable the timer interrupt while it is running. If a set of conditions can be met, this will allow the process to run without regular interference from the timer tick. If other sources of interrupts are directed away from the CPU as well, this process should be able to run uninterrupted for some time. There are a few complications, though. 

Actually going into the tickless mode is relatively easy; the process need only write a nonzero value to `/proc/self/nohz`. The patch imposes a couple of conditions on these processes: (1) the process must be bound to the CPU it is running on, and (2) no other process can be running in the tickless mode on that CPU. If those conditions hold, the write to `/proc/self/nohz` will succeed and the kernel will try to disable the timer tick while that process runs. 

The key word here is "try"; there are a number of things which can keep the disabling of the tick from happening. The first of those is any sort of contention for the CPU. If any other processes are trying to run on the same CPU, the scheduler tick must happen as usual so that decisions on preemption can be made. Since a process can be made runnable from anywhere in the system, Frederic's patch performs a potentially expensive inter-processor interrupt whenever the second process is made runnable on any CPU, regardless of whether that CPU is currently running in the no-tick mode or not. 

Another thing that can gum up the works is read-copy-update (RCU). If there are any RCU callbacks which need to be processed on the CPU, that CPU will not go into the no-tick mode. RCU also needs to be notified whenever the CPU goes into a "quiescent state," so that it can know when it is safe to invoke RCU callbacks on other CPUs. If RCU has indicated an interest in knowing when the target CPU goes quiescent, once again, no-tick mode cannot be entered. The CPU can also be forced out of the no-tick mode if RCU develops a curiosity about quiescent states anywhere in the system. 

Given that RCU is heavily used in contemporary kernels, one would think that its needs would prevent no-tick mode most of the time. Another part of the patch set tries to mitigate that problem with the realization that, if a process is running in user space with the timer tick disabled, the associated CPU is necessarily quiescent. When a CPU is running in this mode, it will enter an "extended quiescent state" which eliminates the need for notification to the rest of the system. The extended quiescent state will probably increase the amount of no-tick time on a processor considerably, but at a small cost: the architecture-level code must add hooks to notify the no-tick code on every kernel entry and exit. 

Reviews of the code, so far, have focused on various details which need to be managed differently, but there has not been a lot of criticism of the concept. It's early-stage code, so it doesn't take care of everything that normally happens during the timer tick, a fact which reviewers have pointed out. The biggest gripe, perhaps, has to do with the conditions mentioned at the beginning of the article: the process must be bound to a single CPU, and there can only be one no-tick process running on that CPU. Peter Zijlstra [said](<https://lwn.net/Articles/420548/>): 

Well yes, this interface of explicitly marking a task and cpu as task_no_hz is kinda restrictive and useless. When I run 4 cpu-bound tasks on a quad-core I shouldn't have to do anything to benefit from this. 

Frederic has indicated that the code can be changed to lift those restrictions, but at the cost of some added complexity. Once the restrictions are gone, it may make sense to just enable the no-tick mode whenever the workload is right for it, regardless of a request (or the lack thereof) from any specific process. That would make the no-tick mode more generally useful; it would also reduce the role of the timer tick just a little more. The kernel would still be far from a fully tickless system, but every step in that direction helps.
