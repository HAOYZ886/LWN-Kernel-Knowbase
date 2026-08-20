---
title: "KS2008: All about threads"
url: https://lwn.net/Articles/298840/
date: "September 17, 2008"
category: Interrupts
author: "By Jonathan Corbet September 17, 2008 LWN's 2008 Kernel Summit coverage"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 17, 2008

* * *

LWN's [2008 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2008/>)

Ben Herrenschmidt led a session on the management of thread pools in the kernel. Kernel threads are typically used as a way for kernel code to do long-running work (which might sleep) as a separate task. The main mechanism used in the kernel now is the workqueue interface, but workqueues are not perfect. They have become a sort of last resort for all kinds of tasks which need to run in process context. 

Problems with workqueues include the fact that they serialize all tasks, even when that serialization is not needed. In some cases, this serialization could lead to deadlocks. Workqueues offer developers the choice of setting up their own dedicated worker threads or using keventd - a set of per-CPU threads shared across all users. The dedicated threads are often overkill for the developer's needs, but using keventd can lead to unpredictable latencies. Often there is no good choice. What's needed is an API that can allow more than one thing to happen on any given CPU while still providing shared threads and low latency. 

One idea is to allow keventd to fork. There could be a new form of workqueue with an "asynchronous" flag set. When a task is queued, keventd would fork and process the task immediately. It would be a relatively easy change to make, but it would also be somewhat inefficient - forks are expensive. 

Another option would be to go with one of the existing thread pool implementations; there are already a few in circulation. The pdflush daemon has a simple mechanism which can grow and shrink the pool of threads based on demand. Btrfs has a thread pool which is tightly tailored to its needs; it does not resize the pool, but it does provide low latency. The sunrpc code has a thread pool which Ben described as "scary." There is also a proposal from David Howells for a "slow work" mechanism. It is the most generic of the options, and supports resizing as well. 

The options were discussed for a bit; Linus's suggestion at the end was to just extend the workqueue interface to provide a small, fixed-size pool. Ben replied that the code for resizing the pool is sufficiently simple that there is no point in leaving it out. 

Thomas Gleixner led a discussion on a related subject: the threaded interrupt handlers which are currently living in the realtime tree. It seems that the realtime developers have finally recovered from having taken on the maintainership of the x86 code and are now getting back to thinking about getting the remaining realtime code merged. 

The realtime tree is set up to thread almost all interrupt handlers, but that will not work for the mainline. Some devices will continue to run with synchronous interrupt handling, and the idea of running software interrupts in threads is not popular with the networking developers. So the suggestion is to provide a new version of `request_irq()` which would allow a driver to set up a threaded interrupt handler. In the absence of a change by the driver maintainer, interrupt handlers would continue to be run synchronously. 

Linus strongly requested that a new request function be added, rather than making a change to `request_irq()` itself. It seems he is still feeling the pain of previous changes to `request_irq()`, which have required fixing massive numbers of drivers. The separate request function was always in the plan; the requirements are significantly different. In particular, drivers using threaded interrupt handlers still need to provide a small, synchronous handler which can determine whether the driver's device is actually interrupting. Without that small handler, it is hard to make the handling of shared interrupt lines work right. 

There was some discussion of details, but no real objection to the overall plan. So chances are good that threaded interrupt handlers will be posted for the 2.6.28 or 2.6.29 development cycles.
