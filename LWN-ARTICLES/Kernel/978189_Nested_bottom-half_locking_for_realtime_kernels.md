---
title: "Nested bottom-half locking for realtime kernels"
url: https://lwn.net/Articles/978189/
date: "June 17, 2024"
category: "Interrupts-Software; Realtime; Releases-6.11"
author: "By Jonathan Corbet June 17, 2024"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 17, 2024

Software-interrupt handlers (also called "bottom halves") have a long history in the Linux kernel; for much of that history, developers have wished that they could go away. One of their unfortunate characteristics is that they can add unexpected latency to the execution of unrelated processes; this problem is felt especially acutely in the realtime-preemption community. The solution adopted there has created problems of its own, though; in response Sebastian Andrzej Siewior is [proposing](<https://lwn.net/ml/all/20240612170303.3896084-1-bigeasy@linutronix.de>) a new locking mechanism for realtime builds of the kernel that may have benefits for non-realtime users as well. 

In normal kernel builds, a software-interrupt handler will run, if needed, at the earliest opportunity that the kernel finds; usually, that is immediately after the completion of a hardware-interrupt handler or on return from the kernel to user space. Either way, software-interrupt handling can delay the execution of a process that may have nothing to do with the creation of that interrupt. For most systems, that delay is not usually a problem, but realtime kernels are all about response time; a badly timed software-interrupt handler has the potential to cause a realtime task to miss its deadline. 

It turns out that the realtime developers are firmly of the opinion that they have not worked on that project for over two decades just to be thwarted by a software-interrupt handler. So those handlers have been made preemptible like nearly everything else in realtime kernels. That change only addresses part of the problem, though. The kernel makes heavy use of per-CPU variables as a way of avoiding contention between processors; as long as no other CPU can access a memory location, there will be no contention for it, and no need for locking to ensure mutual exclusion. Except, of course, if a software-interrupt handler runs and tries to access the same data. 

To avoid such problems, kernel code can call `local_bh_disable()`, which blocks the execution of software-interrupt handlers until `local_bh_enable()` is called. A call to `local_bh_disable()` will also disable preemption and migration for the running task, ensuring that it has sole access to the CPU during its critical section. That solves the problem of racing with software-interrupt handlers (or any other kernel code) on the same CPU, but creates another latency problem for realtime kernels; as long as preemption is disabled, a higher-priority process cannot run on that CPU, once again threatening to increase latency for that higher-priority process. 

The solution to _that_ problem in the realtime tree is to make tasks preemptible while software-interrupt handlers are disabled. But, since a task may be depending on `local_bh_disable()` to keep other tasks from accessing its per-CPU data, `local_bh_disable()` takes a per-CPU lock on realtime kernels. As a result, only one task can be running with software interrupts disabled on any given CPU at a time. 

But, it almost goes without saying, there is _another_ problem. If a low-priority process enters a `local_bh_disable()` section, it can be preempted within that section and prevented from executing (and restoring software interrupts) for a long time. That could block a higher-priority process from completing a `local_bh_disable()` call of its own. It is, in other words, a classic priority-inversion situation. Here, the problem is worsened by the fact that, in all likelihood, there is no actual contention between the two tasks; they are probably calling `local_bh_disable()` to protect entirely different data. 

This situation highlights a problem with disabling software interrupt handling: it is essentially a big lock that provides no indication of what data it is actually protecting. That, in turn, points to a potential solution: replace the big lock with fine-grained locking that protects a limited and well-defined set of data. That is the approach taken by Siewior's patch set. Specifically, it adds a pair of new macros: 

```
local_lock_nested_bh(local_lock_t *lock);
        local_unlock_nested_bh(local_lock_t *lock);
```

Using this mechanism requires auditing each `local_bh_disable()` section, figuring out which data is protected therein, and adding a [`local_lock_t`](<https://docs.kernel.org/locking/locktypes.html#cpu-local-locks>) (a specialized lock that only prevents access from the same CPU) to that data structure. That lock can then be passed to `local_lock_nested_bh()` to protect only that structure while not blocking concurrent execution by unrelated code. 

Code using this approach must still call `local_bh_disable()` to prevent access by software-interrupt handlers and to prevent migration to another CPU. But, once all of the `local_bh_disable()` sections have been audited and adjusted (a job that is reminiscent of [the long effort to remove the Big Kernel Lock](<https://lwn.net/Kernel/Index/#Big_kernel_lock>)), it will be possible to remove the lock that realtime kernels take in `local_bh_disable()`, eliminating a significant source of contention and latency. Benchmark results posted with the patch series show a significant improvement (14.5%) for a networking workload once that lock is removed. 

For non-realtime kernels, instead, `local_lock_nested_bh()` is essentially a no-op, though it does provide information to the locking checker for debugging purposes. Local locks have no effect on non-realtime kernels, and do not require any storage. Thus, this patch satisfies one of the rules that has constrained realtime development from the start: realtime-specific features must not have a performance impact on non-realtime kernels. 

This work will have a significant benefit for the rest of the kernel, though, even if it doesn't change the generated code in most cases. With the current `local_bh_disable()` pattern, there is no indication of what data is being protected. That makes it hard to reason about concurrent access, and makes the introduction of bugs more likely. Once this work is done, the locking rules for the affected data structures will be documented; in many cases, that may make it possible to stop disabling software interrupts entirely in favor of a more focused locking scheme. 

This patch set is in its sixth revision. Previous postings have resulted in some significant changes, mostly in how some networking subsystems were changed to use the new mechanism; the core concept has remained mostly the same. A few developers have indicated their acceptance of this work, so chances are good that it will find its way upstream before too long.
