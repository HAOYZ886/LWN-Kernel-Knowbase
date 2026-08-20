---
title: Working on workqueues
url: https://lwn.net/Articles/403891/
date: "September 7, 2010"
category: Kernel threads; Workqueues
author: "By Jonathan Corbet September 7, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 7, 2010

One of the biggest internal changes in 2.6.36 will be the adoption of [concurrency-managed workqueues](<http://lwn.net/Articles/355700/>). The short-term goal of this work is to reduce the number of kernel threads running on the system while simultaneously increasing the concurrency of tasks submitted to workqueues. To that end, the per-workqueue kernel threads are gone, replaced by a central set of threads with names like `[kworker/0:0]`; workqueue tasks are then dispatched to the threads via an algorithm which tries to keep exactly one task running on each CPU at all times. The result should be better use of the CPU for workqueue tasks and less memory tied up by the workqueue machinery. 

That is a worthwhile result in its own right, but it's really only a beginning. The 2.6.36 workqueue patches were deliberately designed to minimize the impact on the rest of the kernel, so they preserved the existing workqueue API. But the new code is intended to do more than replace workqueues with a cleverer implementation; it is really meant to be a general-purpose task management system for the kernel. Making full use of that capability will require changes in the calling code - and in code which does not yet use workqueues at all. 

In kernels prior to 2.6.36, workqueues are created with `create_workqueue()` and a couple of variants. That function will, among other things, start up one or more kernel threads to handle tasks submitted to that workqueue. In 2.6.36, that interface has been preserved, but the workqueue it creates is a different beast: it has no dedicated threads and really just serves as a context for the submission of tasks. The API is considered deprecated; the proper way to create a workqueue now is with: 

```
int alloc_workqueue(char *name, unsigned int flags, int max_active);
```

The `name` parameter names the queue, but, unlike in the older implementation, it does not create threads using that name. The `flags` parameter selects among a number of relatively complex options on how work submitted to the queue will be executed; its value can include: 

  * **`WQ_NON_REENTRANT`** : "classic" workqueues guaranteed that no task would be run by two threads simultaneously on the same CPU, but made no such guarantee across multiple CPUs. If it was necessary to ensure that a task could not be run simultaneously anywhere in the system, a single-threaded workqueue had to be used, possibly limiting concurrency more than desired. With this flag, the workqueue code will provide that systemwide guarantee while still allowing different tasks to run concurrently. 

  * **`WQ_UNBOUND`** : workqueues were designed to run tasks on the CPU where they were submitted in the hope that better memory cache behavior would result. This flag turns off that behavior, allowing submitted tasks to be run on any CPU in the system. It is intended for situations where the tasks can run for a long time, to the point that it's better to let the scheduler manage their location. Currently the only user is the object processing code in the FS-Cache subsystem. 

  * **`WQ_FREEZEABLE`** : this workqueue will be frozen when the system is suspended. Clearly, workqueues which can run tasks as part of the suspend/resume process should not have this flag set. 

  * **`WQ_RESCUER`** : this flag marks workqueues which may be involved in memory reclaim; the workqueue code responds by ensuring that there is always a thread available to run tasks on this queue. It is used, for example, in the ATA driver code, which always needs to be able to run its I/O completion routines to be sure it can free memory. 

  * **`WQ_HIGHPRI`** : tasks submitted to this workqueue will put at the head of the queue and run (almost) immediately. Unlike ordinary tasks, high-priority tasks do not wait for the CPU to become available; they will be run right away. That means that multiple tasks submitted to a high-priority queue may contend with each other for the processor. 

  * **`WQ_CPU_INTENSIVE`** : tasks on this workqueue can be expected to use a fair amount of CPU time. To keep those tasks from delaying the execution of other workqueue tasks, they will not be taken into account when the workqueue code determines whether the CPU is available or not. CPU-intensive tasks will still be delayed themselves, though, if other tasks are already making use of the CPU. 

The combination of the `WQ_HIGHPRI` and `WQ_CPU_INTENSIVE` flags takes this workqueue out of the concurrency management regime entirely. Any tasks submitted to such a workqueue will simply run as soon as the CPU is available. 

The final argument to `alloc_workqueue()` (we _are_ still talking about `alloc_workqueue()`, after all) is `max_active`. This parameter limits the number of tasks which can be executing simultaneously from this workqueue on any given CPU. The default value (used if `max_active` is passed as zero) is 256, but the actual maximum is likely to be far lower, given that the workqueue code really only wants one task using the CPU at any given time. Code which requires that workqueue tasks be executed in the order in which they are submitted can use a `WQ_UNBOUND` workqueue with `max_active` set to one. 

(Incidentally, much of the above was cribbed from [Tejun Heo's in-progress document](<https://lwn.net/Articles/403918/>) on workqueue usage). 

The long-term plan, it seems, is to convert all `create_workqueue()` users over to an appropriate `alloc_workqueue()` call; eventually `create_workqueue()` will be removed. That task may take a little while, though; a quick grep turns up nearly 300 call sites. 

An even longer-term plan is to merge a number of other kernel threads into the new workqueue mechanism. For example, the block layer maintains a set of threads with names like `flush-8:0` and `bdi-default`; they are charged with getting data written out to block devices. Tejun recently [posted a patch](<http://lwn.net/Articles/403653/>) to replace those threads with workqueues. This patch has made some developers a little nervous - problems with writeback could create no end of trouble when the system is under memory pressure. So it may be slow to get into the mainline, but it will probably get there eventually unless regressions turn up. 

After that, there is no end of special-purpose kernel threads elsewhere in the system. Not all of them will be amenable to conversion to workqueues, but quite a few of them should be. Over time, that should translate to less system resource use, cleaner "`ps`" output, and a better-running system.
