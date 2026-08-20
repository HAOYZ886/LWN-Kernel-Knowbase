---
title: Local locks in the kernel
url: https://lwn.net/Articles/828477/
date: "August 11, 2020"
category: "Locking mechanisms-Local locks; Realtime"
author: "August 11, 2020 This article was contributed by Marta Rybczyńska"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

August 11, 2020

This article was contributed by Marta Rybczyńska

The Linux kernel has never lacked for synchronization primitives and locking mechanisms, so one might justifiably wonder why there might be a need to add another one. The addition of [local locks](<https://lwn.net/ml/linux-kernel/20200527201119.1692513-1-bigeasy@linutronix.de/>) to 5.8 provides an answer to that question. These locks, which have their origin in the realtime (`PREEMPT_RT`) tree, were created to solve some realtime-specific problems, but they also bring some much-needed structure to a common locking pattern used in non-realtime kernels as well. 

#### Lock types in Linux

The Linux kernel offers developers a number of lock types to choose from. They could be roughly divided, until recently, into two categories: spinning and sleeping locks.

A kernel function attempting to acquire a spinning lock that is owned by another thread will spin (loop actively) until the other thread, which must be running on a different CPU, releases the lock. This type of lock is fast, but it may waste CPU cycles if the wait lasts for a long time. Spinning locks are thus used around short sections of code. Longer code sections protected by spinning locks will increase the overall system latency; code that needs to respond to an event quickly may be blocked on a lock. The category of spinning locks contains spinlocks and read-write locks.

The situation is different with sleeping locks; a thread taking such a lock will, as the name suggests, relinquish the CPU if it cannot obtain the lock. This type of lock works for longer sections of critical code, but takes a longer time to obtain. Also, sleeping locks cannot be taken when a thread is running in atomic context; that happens, for example, when interrupts are disabled, the code holds a spinlock, or it holds an [atomic kmap](<https://lwn.net/Articles/29564/>) (atomic kernel mapping). In non-`PREEMPT_RT` kernels, sleeping locks include all types of mutexes and semaphores. In practice, even sleeping locks do spinning in some cases if there is a possibility to obtain the lock rapidly. For example, mutexes may spin if the mutex owner is running (and thus should release the mutex shortly). This is called "opportunistic spinning"; interested readers can look into the details in the [kernel documentation](<https://www.kernel.org/doc/html/latest/locking/mutex-design.html>).

The implementation of many lock types changes in the `PREEMPT_RT` kernels; in particular, most spinning locks becoming sleeping locks. 

#### Disabling interrupts and preemption as locking

While not called "locking", another way of serializing access to certain types of data exists in practice, usually used in low-level code; it works by disabling interrupts, preemption, or both. These actions only apply to the running CPU. 

The Linux kernel is preemptive, meaning that a task can be stopped at (almost) any moment to give the CPU to a higher-priority task. Tasks may be also moved to a different CPU at almost any time. Some code sections, usually those dealing with per-CPU data, need to ensure that they run continuously on the same CPU without interference from other tasks. This code may not need a global lock; since it only needs to modify per-CPU data, there should be no possibility of concurrent access from elsewhere. Such code can simply disable preemption with `preempt_disable()`, restoring it with `preempt_enable()`. If the goal is to use per-CPU data, additional helper functions exist; `get_cpu()` disables preemption and returns the current CPU ID, and `put_cpu()` enables preemption.

Interrupts may be delivered to the CPU while a task is executing; that too may cause unexpected concurrent access to per-CPU data. To prevent this problem, the developer can disable interrupt delivery with `local_irq_disable()` and then enable it with `local_irq_enable()`. If the code is running in a context where the interrupts might be already disabled, they should use `local_irq_save()` and `local_irq_restore()`; this variant saves and restores the previous status in addition to disabling or enabling the interrupts. It is worth noting that disabling interrupts also disables preemption. While interrupts are disabled, the code is running in atomic context and the developers need to be careful to avoid, among other things, any operations that may sleep or call into the scheduler.

These conventions for disabling preemption and interrupts are somewhat problematic. When a developer creates a lock, they (should) define exactly what is protected by that lock. Development tools like the lockdep checker can then help to ensure that the locking rules are followed in all cases. There _is_ no lock when preemption disabling is used, though, and often no clear idea of what is being protected, making it easy for bugs to slip in. 

The problems become more acute in the realtime setting. The key goal in a realtime kernel is that nothing must prevent the highest-priority task from running immediately. Disabling preemption can prevent the kernel from making the CPU available to a process that needs it, and disabling interrupts can delay the arrival of events that must be responded to as quickly as possible. The realtime developers have duly worked for many years to prevent disabling either of those things whenever an alternative exists. 

#### Introducing local locks

The solution to these problems is to create a new type of explicit lock, called a "local lock". On non-realtime systems, the acquisition of a local lock simply maps to disabling preemption (and possibly interrupts). On realtime systems, instead, local locks are actually sleeping spinlocks; they do not disable either preemption or interrupts. They are sufficient to serialize access to the resource being protected without increasing latencies in the system as a whole. 

The local-lock operations are defined as macros and they use the `local_lock_t` type which, on non-realtime systems, only contains useful data when lock debugging is enabled. The developer can initialize a lock using `local_lock_init()`, then take the lock using `local_lock()` and release it using `local_unlock()`. To also disable interrupts, the developer can use `local_lock_irq()` and `local_unlock_irq()`. Finally, another set of functions allows disabling interrupts while remembering their previous state: `local_lock_irqsave()` and `local_unlock_irqrestore()`. 

Local lock operations map into the preemption and interrupt disabling in the following way: 

```
local_lock(&llock)             preempt_disable()
         local_unlock(&llock)           preempt_enable()
         local_lock_irq(&llock)         local_irq_disable()
         local_unlock_irq(&llock)       local_irq_enable()
         local_lock_save(&llock)        local_irq_save()
         local_unlock_restore(&llock)   local_irq_restore()
```

The advantages of local locks for non-`PREEMPT_RT` configurations is that they clarify what is actually being protected and they allow for lock debugging, including static analysis and runtime debugging with lockdep. This was not possible with direct calls to disable preemption and interrupts. Having a clear scope allows better documentation of locking in the code, with different local locks used in different parts of the code, instead of one set of context-less functions. In the realtime world, additionally, code holding a local lock can still be preempted if need be, preventing it from blocking a higher-priority task. 

With the addition of local locks, the lock nesting rules change somewhat. Spinning locks can nest (when all other locking rules are met) into all other types of locks (spinning, local, or sleeping). Local locks can nest into sleeping locks.

#### Summary

Local locks have been merged for the 5.8 kernel, which means they are available now. This is a welcome addition, as it makes per-CPU locking have the same semantics as the other locks. This should avoid some errors, and allow lockdep in contexts where it was not possible to use before. In addition, it will make the merging of the remaining `PREEMPT_RT` patches that much easier.
