---
title: Controlling realtime priorities in kernel threads
url: https://lwn.net/Articles/818388/
date: "April 23, 2020"
category: "Modules-Exported symbols; Realtime; Scheduler-Realtime"
author: "By Jonathan Corbet April 23, 2020"
---

By **Jonathan Corbet**  
April 23, 2020

The realtime scheduler classes are intended to allow a developer to state which tasks have the highest priorities with the assurance that, at any given time, the highest-priority task will have unimpeded access to the CPU. The kernel itself carries out a number of tasks that have tight time constraints, so it is natural to want to assign realtime priorities to kernel threads carrying out those tasks. But, as Peter Zijlstra [argues](<https://lwn.net/ml/linux-kernel/20200422112719.826676174@infradead.org/>) in a new patch set, it makes little sense for the kernel to be assigning such priorities; to put an end to that practice, he is proposing to take away most of the kernel's ability to prioritize its own threads. 

In the classic realtime model, there are two scheduling classes: `SCHED_FIFO` and `SCHED_RR`. Processes in either class have a simple integer priority. `SCHED_FIFO` processes run until they voluntarily give up the CPU, with the highest-priority process going first. `SCHED_RR`, instead, rotates through all runnable processes at the highest priority level, giving each a fixed time slice. In either class, processes with a lower realtime priority will be completely blocked until all higher-priority processes are blocked, and processes in either class will, regardless of priority level, run ahead of normal, non-realtime work in the `SCHED_NORMAL` class. 

The kernel pushes a large (and increasing) amount of work out into kernel threads, which are special processes running within the kernel's address space. This is done to allow that work to happen independently of any other thread of execution, under the control of the system scheduler. Most kernel threads run in the `SCHED_NORMAL` class and must contend with ordinary user-space processes for CPU time. Others, though, are deemed special enough that they should run ahead of user-space work; one way to make that happen is to put those threads into the `SCHED_FIFO` class. 

But then a question arises: which priority should any given thread have? Answering that question requires judging the importance of a given thread relative to all of the other threads running at realtime priority — and relative to any user-space realtime work as well. That is going to be a difficult question to answer, even if the answer turns out to be the same for every system and workload, which seems unlikely. In general, kernel developers don't even try; they just pick something. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Zijlstra believes that this exercise is pointless: ""the kernel has no clue what actual priority it should use for various things, so it is useless (or worse, counter productive) to even try"". So he has changed the kernel's internal interfaces to take away the ability to run at a specific `SCHED_FIFO` priority. What remains is a set of three functions: 

```
void sched_set_fifo(struct task_struct *p);
        void sched_set_fifo_low(struct task_struct *p);
        void sched_set_normal(struct task_struct *p, int nice);
```

For loadable modules, these become the only functions available for manipulating a thread's scheduling information. All three functions are exported only to modules with GPL-compatible licenses. A call to `sched_set_fifo()` puts the given process into the `SCHED_FIFO` class at priority 50 — halfway between the minimum and maximum values. For threads with less pressing requirements, `sched_set_fifo_low()` sets the priority to the lowest value (one) instead. Calling `sched_set_normal()` returns the thread to the `SCHED_NORMAL` class with the given `nice` value. 

The bulk of the patch set consists of changes to specific subsystems to make them use the new API; it gives a picture of how current kernels are handling `SCHED_FIFO` threads now. Here's what turns up: 

> Subsystem| Priority| Description  
> ---|---|---  
> [Arm bL switcher](<https://lwn.net/ml/linux-kernel/20200422112831.339113081@infradead.org/>) | 1 | The Arm big.LITTLE switcher thread  
> [crypto](<https://lwn.net/ml/linux-kernel/20200422112831.397571979@infradead.org/>) | 50 | Crypto engine worker thread  
> [ACPI](<https://lwn.net/ml/linux-kernel/20200422112831.455977635@infradead.org/>) | 1 | ACPI processor aggregator driver  
> [drbd](<https://lwn.net/ml/linux-kernel/20200422112831.515522261@infradead.org/>) | 2 | Distributed, replicated block device request handling  
> [PSCI checker](<https://lwn.net/ml/linux-kernel/20200422112831.574539982@infradead.org/>) | 99 | PSCI firmware hotplug/suspend functionality checker  
> [msm](<https://lwn.net/ml/linux-kernel/20200422112831.633583714@infradead.org/>) | 16 | MSM GPU driver  
> [DRM](<https://lwn.net/ml/linux-kernel/20200422112831.692776147@infradead.org/>) | 1 | Direct rendering request scheduler  
> [ivtv](<https://lwn.net/ml/linux-kernel/20200422112831.752048390@infradead.org/>) | 99 | Conexant cx23416/cx23415 MPEG encoder/decoder driver  
> [mmc](<https://lwn.net/ml/linux-kernel/20200422112831.811655681@infradead.org/>) | 1 | MultiMediaCard drivers  
> [cros_ec_spi](<https://lwn.net/ml/linux-kernel/20200422112831.870192415@infradead.org/>) | 50 | ChromeOS embedded controller SPI driver  
> [powercap](<https://lwn.net/ml/linux-kernel/20200422112831.928975262@infradead.org/>) | 50 | "Powercap" idle-injection driver  
> [powerclamp](<https://lwn.net/ml/linux-kernel/20200422112832.046704918@infradead.org/>) | 50 | Intel powerclamp thermal management subsystem  
> [sc16is7xx](<https://lwn.net/ml/linux-kernel/20200422112832.105778342@infradead.org/>) | 50 | NXP SC16IS7xx serial port driver  
> [watchdog](<https://lwn.net/ml/linux-kernel/20200422112832.165197058@infradead.org/>) | 99 | Watchdog timer driver subsystem  
> [irq](<https://lwn.net/ml/linux-kernel/20200422112832.224090625@infradead.org/>) | 50 | Threaded interrupt handling  
> [locktorture](<https://lwn.net/ml/linux-kernel/20200422112832.283981577@infradead.org/>) | 99 | Locking torture-testing module  
> [rcuperf](<https://lwn.net/ml/linux-kernel/20200422112832.343851342@infradead.org/>) | 1 | Read-copy-update performance tester  
> [rcutorture](<https://lwn.net/ml/linux-kernel/20200422112832.403795265@infradead.org/>) | 1 | Read-copy-update torture tester  
> [sched/psi](<https://lwn.net/ml/linux-kernel/20200422112832.465990342@infradead.org/>) | 1 | Pressure-stall information data gathering  
  
As one can see, there is indeed a fair amount of variety in the priority values chosen by kernel developers for their threads. Additionally, the drbd driver was using the `SCHED_RR` class for reasons that weren't entirely clear. After Zijlstra's patch set is applied, all of the subsystems using a priority of one have been converted to use `sched_set_fifo_low()`, while the rest use `sched_set_fifo()`, giving them all a priority of 50\. 

There have been responses to a number of the patches thus far, mostly offering Reviewed-by tags or similar. It seems that few, if any, kernel developers are strongly attached to the `SCHED_FIFO` priority values that they chose when they had to come up with a number to put into that structure field. It is thus unlikely that there is going to be any sort of serious opposition to this patch set going in. 

The end result is not limited to a rationalization of `SCHED_FIFO` values inside the kernel, though. One of the objections Zijlstra raises about `SCHED_FIFO` in general is that, even if a developer is able to choose perfect priority values for their workload, all that work goes by the wayside if that workload has to be combined with another, which will have its own set of priority values. The chances of those two sets of values combining into a coherent whole are relatively small. 

In current kernels, _every_ realtime workload using `SCHED_FIFO` faces this problem, since the priority choices made for that workload have to be combined with the choices made for kernel threads — choices that have not really been thought through and which are not documented anywhere. Making the kernel's configuration for `SCHED_FIFO` priorities predictable should make life easier for realtime system designers, who are unlikely to mind having fewer variables to worry about.
