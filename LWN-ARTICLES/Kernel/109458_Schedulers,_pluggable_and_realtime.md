---
title: "Schedulers, pluggable and realtime"
url: https://lwn.net/Articles/109458/
date: "November 3, 2004"
category: Realtime; Scheduler
author: 
---

A constant fact of Linux kernel development would appear that people always want to play around with the CPU scheduler. Con Kolivas (with help from William Lee Irwin) has decided to make this playing easier through the creation of a [pluggable scheduler framework](<https://lwn.net/Articles/109049/>). This mechanism is intended to make it possible for multiple schedulers to exist in the kernel, with one being selected for use at boot time. With "plugsched" in place, developers interested in experimenting with schedulers could switch quickly between them while running the same kernel. 

The patch works by splitting the large body of code in `kernel/sched.c` into public and private parts. Code meant to be shared between schedulers goes into a new `scheduler.c` file, while the current (and default) scheduler stays put. Also added to `scheduler.c` is a new structure (`struct sched_drv`) containing pointers to the functions which handle scheduling tasks. These functions are invoked for various process events (`fork()`, `exit()`, etc.), to obtain scheduling-related information, and, of course, for calls to the core `schedule()` function. Implementing a new scheduler is simply a matter of writing replacements for the relevant functions and plugging the whole thing in. 

There have been few objections to the pluggable scheduler implementation. Ingo Molnar, however, [is strongly opposed to the idea](<https://lwn.net/Articles/109460/>) in the first place: 

I believe that by compartmenting in the wrong way we kill the natural integration effects. We'd end up with 5 (or 20) bad generic schedulers that happen to work in one precise workload only, but there would not be enough push to build one good generic scheduler, because the people who are now forced to care about the Linux scheduler would be content about their specialized schedulers. 

Ingo's position is that having one core scheduler forces developers to think about the whole problem, rather than one small piece of it. In particular, claims Ingo, the [scheduling domains](<https://lwn.net/Articles/80911/>) patch would never have come about if the kernel had pluggable schedulers; instead there would be a separate NUMA scheduler, an SMP scheduler, and so on. 

Ingo, meanwhile, continues his efforts to make the One Big Scheduler provide real-time response. The latest patch is [-RT-2.6.10-rc1-mm2-V0.7.1](<https://lwn.net/Articles/109439/>). The biggest change in recent times is a new semaphore/mutex implementation which sticks closer to the original Linux semaphore semantics; this change allows a number of patches switching parts of the kernel over to the completion interface to be dropped. 

The new semaphores also include a priority inheritance mechanism. Whenever a process blocks on a semaphore, the kernel checks to see if that process has a higher priority than the process currently holding the semaphore. If so, the holder's priority is bumped up to match that of the blocking process. This technique should help to avoid situations where a low-priority process can keep higher-priority tasks from running for extended periods of time.
