---
title: Optimizing preemption
url: https://lwn.net/Articles/563185/
date: "August 14, 2013"
category: Preemption; Voluntary preemption
author: "By Jonathan Corbet August 14, 2013"
---

By **Jonathan Corbet**  
August 14, 2013

The kernel's lowest-level primitives can be called thousands of times (or more) every second, so, as one might expect, they have been ruthlessly optimized over the years. To do otherwise would be to sacrifice some of the system's performance needlessly. But, as it happens, hard-won performance can slip away over the years as the code is changed and gains new features. Often, such performance loss goes unnoticed until a developer decides to take a closer look at a specific kernel subsystem. That would appear to have just happened with regard to how the kernel handles preemption. 

#### User-space access and voluntary preemption

In this case, things got started when Andi Kleen decided to make the user-space data access routines — `copy_from_user()` and friends — go a little faster. As he explained in [the resulting patch set](<https://lwn.net/Articles/562936/>), those functions were once precisely tuned for performance on x86 systems. But then they were augmented with calls to functions like `might_sleep()` and `might_fault()`. These functions initially served in a debugging role; they scream loudly if they are called in a situation where sleeping or page faults are not welcome. Since these checks are for debugging, they can be turned off in a production kernel, so the addition of these calls should not affect performance in situations where performance really matters. 

But, then, in 2004, core kernel developers started to take latency issues a bit more seriously, and that led to an interest in preempting execution of kernel code if a higher-priority process needed the CPU. The problem was that, at that time, it was not exactly clear when it would be safe to preempt a thread in kernel space. But, as Ingo Molnar and Arjan van de Ven noticed, calls to `might_sleep()` were, by definition, placed in locations where the code was prepared to sleep. So a `might_sleep()` call had to be a safe place to preempt a thread running in kernel mode. The result was the [voluntary preemption patch set](<https://lwn.net/Articles/93604/>), adding a limited preemption mode that is still in use today. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The problem, as Andi saw it, is that this change turned `might_sleep()` and `might_fault()` into a part of the scheduler; it is no longer compiled out of a kernel if voluntary preemption is enabled. That, in turn, has slowed down user-space access functions by (on his system) about 2.5µs for each call. His patch set does a few things to try to make the situation better. Some functions (`should_resched()`, which is called from `might_sleep()`, for example) are marked `__always_inline` to remove the function calling overhead. A new `might_fault_debug_only()` function goes back to the original intent of `might_fault()`; it disappears entirely when it is not needed. And so on. 

Linus had no real objection to these patches, but they clearly raised a couple of questions in his mind. One of his first comments was [a suggestion](<https://lwn.net/Articles/563187/>) that, rather than optimizing the `might_fault()` call in functions like `copy_from_user()`, it would be better to omit the check altogether. Voluntary preemption points are normally used to switch between kernel threads when an expensive operation is being performed. If a user-space access succeeds without faulting, it is not expensive at all; it is really just another memory fetch. If, instead, it causes a page fault, there will already be opportunities for preemption. So, Linus reasoned, there is little point in slowing down user-space accesses with additional preemption checks. 

#### The problem with full preemption

To this point, the discussion was mostly concerned about voluntary preemption, where a thread running in the kernel can lose access to the processor, but only at specific spots. But the kernel also supports "full preemption," which allows preemption almost anywhere that preemption has not been explicitly disabled. In the early days of kernel preemption, many users shied away from the full preemption option, fearing subtle bugs. They may have been right at the time, but, in the intervening years, the fully preemptible kernel has become much more solid. Years of experience, helped by tools like the locking validator, can work wonders that way. So there is little reason to be afraid to enable full preemption at this point. 

With that history presumably in mind, H. Peter Anvin entered the conversation with [a question](<https://lwn.net/Articles/563188/>): should voluntary preemption be phased out entirely in favor of full kernel preemption? It turns out that there _is_ still one reason to avoid turning on full preemption: as Mike Galbraith [put it](<https://lwn.net/Articles/563189/>), ""PREEMPT munches throughput"". Complaints about the cost of full preemption have been scarce over the years, but, evidently, it does hurt in some cases. As long as there is a performance penalty to the use of full preemption, it is going to be hard to convince throughput-oriented users to switch to it. 

There would not seem to be any fundamental reason why full preemption should adversely affect throughput. If the rate of preemption were high, there could be some associated cache effects, but preemption should be a relatively rare event in a throughput-sensitive system. That suggests that something else is going on. A clue about that "something else" can be found in Linus's [observation](<https://lwn.net/Articles/563190/>) that the testing of the preemption count — which happens far more often in a fully preemptible kernel — is causing the compiler to generate slower code. 

The thing is, even if that is almost never taken, just the fact that there is a conditional function call very often makes code generation *much* worse. A function that is a leaf function with no stack frame with no preemption often turns into a non-leaf function with stack frames when you enable preemption, just because it had a RCU read region which disabled preemption. 

So configuring full preemption into the kernel can make performance-sensitive code slower. Users who are concerned about latency may well be willing to make that tradeoff, but those who want throughput will not be so agreeable. The good news is that it might be possible to do something about this problem and keep both camps happy. 

#### Optimizing full preemption

The root of the problem is accesses to the variable known as the "preemption count," which can be found in the `thread_info` structure, which, in turn lives at the bottom of the kernel stack. It is not just a counter, though; instead it is a 32-bit quantity that has been divided up into several subfields: 

  * The actual preemption count, indicating how many times kernel code has disabled preemption. This counter allows calls like `preempt_disable()` to be nested and still do the right thing (eight bits). 

  * The software interrupt count, indicating how many nested software interrupts are being handled at the moment (eight bits). 

  * The hardware interrupt count (ten bits on most architectures). 

  * The `PREEMPT_ACTIVE` bit indicating that the current thread is being (or just has been) preempted. 

This may seem like a complicated combination of fields, but it has one useful feature: the preemptability of the currently-running thread can be tested by comparing the entire preemption count against zero. If any of the counters has been incremented (or the `PREEMPT_ACTIVE` bit set), preemption will be disabled. 

It seems that the cost of testing this count might be reduced significantly with some tricky assembly language work; that is being hashed out as of this writing. But there's another aspect of the preemption count that turns out to be costly: its placement in the `thread_info` structure. The location of that structure must be derived from the kernel stack pointer, making the whole test significantly more expensive. 

The important realization here is that there is (almost) nothing about the preemption count that is specific to any given thread. It will be zero for every non-executing thread; and no executing thread will be preempted if the count is nonzero. It is, in truth, more of an attribute of the CPU than of the running process. And that suggests that it would be naturally stored as a per-CPU variable. Peter Zijlstra has posted [a patch](<https://lwn.net/Articles/563088/>) that changes things in just that way. The patch turned out to be relatively straightforward; the only twist is that the `PREEMPT_ACTIVE` flag, being a true per-thread attribute, must be saved in the `thread_info` structure when preemption occurs. 

Peter's first patch didn't quite solve the entire problem, though: there is still the matter of the `TIF_NEED_RESCHED` flag that is set in the `thread_info` structure when kernel code (possibly running in an interrupt handler or on another CPU) determines that the currently-running task should be preempted. That flag must be tested whenever the preemption count returns to zero, and in a number of other situations as well; as long as that test must be done, there will still be a cost to enabling full preemption. 

Naturally enough, Linus has [a solution to this problem](<https://lwn.net/Articles/563195/>) in mind as well. The "need rescheduling" flag would move to the per-CPU preemption count as well, probably in the uppermost bit. That raises an interesting problem, though. The preemption count, as a per-CPU variable, can be manipulated without locks or the use of expensive atomic operations. This new flag, though, could well be set by another CPU entirely; putting it into the preemption count would thus wreck that count's per-CPU nature. But Linus has a scheme for dancing around this problem. The "need rescheduling" flag would only be changed using atomic operations, but the remainder of the preemption count would be updated locklessly as before. 

Mixing atomic and non-atomic operations is normally a way to generate headaches for everybody involved. In this case, though, things might just work out. The use of atomic operations for the "need rescheduling" bit means that any CPU can set that bit without corrupting the counters. On the other hand, when a CPU changes its preemption count, there is a small chance that it will race with another CPU that is trying to set the "need rescheduling" flag, causing that flag to be lost. That, in turn, means that the currently executing thread will not be preempted when it should be. That result is unfortunate, in that it will increase latency for the higher-priority task that is trying to run, but it will not generate incorrect results. It is a minor bit of sloppiness that the kernel can get away with if the performance benefits are large enough. 

In this case, though, there appears to be a better solution to the problem. Peter came back with [an alternative approach](<https://lwn.net/Articles/563259/>) that keeps the `TIF_NEED_RESCHED` flag in the `thread_info` structure, but also adds a copy of that flag in the preemption count. In current kernels, when the kernel sets `TIF_NEED_RESCHED`, it also signals an inter-processor interrupt (IPI) to inform the relevant CPU that preemption is required. Peter's patch makes the IPI handler copy the flag from the `thread_info` structure to the per-CPU preemption count; since that copy is done by the processor that owns the count variable, the per-CPU nature of that count is preserved and the race conditions go away. As of this writing, that approach seems like the best of all worlds — fast testing of the "need rescheduling" flag without race conditions. 

Needless to say, this kind of low-level tweaking needs to be done carefully and well benchmarked. It could be that, once all the details are taken care of, the performance gained does not justify the trickiness and complexity of the changes. So this work is almost certainly not 3.12 material. But, if it works out, it may be that much of the throughput cost associated with enabling full preemption will go away, with the eventual result that the voluntary preemption mode could be phased out.
