---
title: Migration disable for the mainline
url: https://lwn.net/Articles/836503/
date: "November 9, 2020"
category: Realtime; Scheduler
author: "By Jonathan Corbet November 9, 2020"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 9, 2020

The realtime developers have been working for many years to create a kernel where the highest-priority task is always able to run without delay. That has meant a long process of finding and fixing situations where high-priority tasks might be blocked from running; one of the persistent problems in this regard has been kernel code that disables preemption. One tool that the realtime developers have reached for is disabling migration (moving a process from one CPU to another) rather than preemption; this approach has not been entirely popular among scheduler developers, though. Even so, the solution would appear to be [this migration-disable patch set](<https://lwn.net/ml/linux-kernel/20201023101158.088940906@infradead.org/>) from scheduler developer Peter Zijlstra. 

One of the key scalability techniques used in the kernel is per-CPU data. System-wide locking is an effective way of protecting shared data, but it can kill performance in a number of ways, even if a given lock is itself not heavily contested. Any data structure that is only accessed by a single CPU does not need to be protected by system-wide locks, avoiding this problem. Thus, for example, the memory allocators maintain per-CPU lists of available memory that can be handed out without interference from the other CPUs on the system. But kernel code can only safely manipulate per-CPU data if it has exclusive access to the CPU; if some other process is able to jump in, it could find (or create) inconsistent per-CPU data structures. The normal way to prevent this from happening is to disable preemption when necessary; it is a cheap operation (setting a flag, essentially) that ensures that a given task will not be interrupted until its work is done. 

Disabling preemption runs afoul of the goals of the realtime developers, who have put so much work into ensuring that any given task _can_ be interrupted if a higher-priority task needs the CPU. As they have worked to remove preemption-disabled regions, they have observed that, often, all that is really needed is to keep tasks from being moved between CPUs while they are accessing per-CPU data, with perhaps some (normally CPU-local) locking as well. See, for example, the [`kmap_local()` work](<https://lwn.net/Articles/836144/>). Disabling migration still allows a process to be preempted, so it does not interfere with the goals of the realtime project — or so those developers hope. 

Disabling migration brings problems of its own, though. The kernel's CPU scheduler is tasked with making the best use of all of the CPUs in the system. If there are N CPUs available, they should be running the N highest-priority tasks at any given time. That goal cannot be achieved without occasionally moving tasks between CPUs; it would be nice if tasks just happened to land on the right processors every time, but the real world is not like that. Depriving the scheduler of the ability to migrate tasks, even for brief periods, thus takes away a tool that is crucial for the overall behavior and throughput of the system. 

As a simple example of what can happen, consider a system with two CPUs and two tasks, of which only the lower-priority task is runnable. That task enters a migration-disabled section at the same time that the high-priority task becomes runnable on the same CPU. The low-priority task will be duly preempted so that the high-priority task can run. That low-priority task still needs CPU time, though, and meanwhile the other CPU is sitting idle. Normally the scheduler would just migrate the low-priority task over to the idle CPU and allow it to continue but, since that task has disabled migration, it remains stuck and unable to run. Migration disable thus differs from preemption disable, which does not risk creating stuck processes in this way. 

So it is not entirely surprising that the migration-disable capability has not been greeted with open arms by mainline scheduler developers. Those same developers, though (and Zijlstra in particular) understand what is driving this work. So, when Thomas Gleixner [posted a migration-disable patch set](<https://lwn.net/ml/linux-kernel/20200917094202.301694311@linutronix.de/>) in September, Zijlstra declined to apply it, but he also went to work to create an alternative that would be acceptable from a scheduling point of view — on realtime kernels, at least. 

The [patch](<https://lwn.net/ml/linux-kernel/20201023102346.818170844@infradead.org/>) adding the core machinery makes it clear in a leading comment that the migration disable feature is ""(strongly) undesired"". It goes on: 

This is a 'temporary' work-around at best. The correct solution is getting rid of the above assumptions and reworking the code to employ explicit per-cpu locking or short preempt-disable regions. 

The end goal must be to get rid of migrate_disable(), alternatively we need a schedulability theory that does not depend on arbitrary migration. 

There are a couple of particularly tricky areas when it comes to making migration disable work properly. One of those, naturally, is CPU hotplug, which has [already shown itself to be a difficult area](<https://lwn.net/Articles/537562/>) in the past. If a CPU is to be removed from the system, one should first migrate all running processes elsewhere to avoid the even trickier problem of irate users. But if some of those processes have disabled migration, that cannot be immediately done. So the hotplug mechanism had to gain a count of how many tasks in each run queue have disabled migration, and to wait until that number drops to zero. 

Then, there is the issue of blocked tasks described above: there may be a CPU available to run a lower-priority task that has been preempted, but the disabling of migration prevents the task from moving to that available CPU. In a truly pathological situation, several preempted tasks could end up stacked on a CPU and unable to migrate while most of the system remains idle. This sort of violation of [work conservation](<https://en.wikipedia.org/wiki/Work-conserving_scheduler>) does not improve the mood of scheduler developers — and they already have a reputation for grumpiness. 

The approach taken to this problem is not a perfect solution (which may not exist), but hopefully it helps. If a CPU's run queue contains a task that is runnable, but which has been preempted by a higher-priority task, the normal response would be to try to migrate the preempted task elsewhere. If migration has been disabled, that cannot happen, obviously. So the scheduler will try, instead, to migrate the running, higher-priority task to get it out of the way. That is not ideal; migration has its costs, including the potential loss of cache locality, that will now be paid by the higher-priority task. Or, as Zijlstra [put it](<https://lwn.net/ml/linux-kernel/20201023102347.499155098@infradead.org/>): ""This adds migration interference to the higher priority task, but restores bandwidth to system that would otherwise be irrevocably lost"". 

Finally, it's worth pointing out that migration disable will be limited to kernels configured for realtime operation. On everything else, a call to `migrate_disable()` will disable preemption, as is done now. So behavior for most users will not change, at least not directly. But this is another important step toward getting the realtime preemption patches fully migrated into the mainline after all these years.
