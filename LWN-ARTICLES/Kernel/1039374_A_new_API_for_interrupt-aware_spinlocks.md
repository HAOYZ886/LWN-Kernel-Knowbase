---
title: "A new API for interrupt-aware spinlocks"
url: https://lwn.net/Articles/1039374/
date: "October 15, 2025"
category: "Development tools-Rust"
author: "By Daroc Alden October 15, 2025 Kangrejos"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Daroc Alden**  
October 15, 2025

* * *

[Kangrejos](<https://lwn.net/Articles/1039334/>)

Boqun Feng spoke at [ Kangrejos 2025](<https://kangrejos.com/>) about adding a frequently needed API for Rust drivers that need to handle interrupts: interrupt-aware spinlocks. Most drivers will need to communicate information from interrupt handlers to main driver code, and this exchange is frequently synchronized with the use of spinlocks. While his first attempts ran into problems, Feng's ultimate solution could help prevent bugs in C code as well, by tracking the number of nested scopes that have disabled interrupts. The [ patch set](<https://lwn.net/ml/all/20251013155205.2004838-1-lyude@redhat.com/>), which contains work from Feng and Lyude Paul, is still under review. 

Code that acquires a spinlock needs to disable preemption: otherwise, if it were preempted, everything else contending for the lock would just pointlessly burn CPU time. The same thing (almost) applies to using spinlocks in interrupts, Feng said. If kernel code acquires a spinlock in process context, and then an interrupt arrives, the handler for which tries to acquire the same lock, the system will deadlock. The simple rule in the kernel is that if a lock is ever used in both interrupt handler and process context, the lock must be "irq-safe" (meaning that the acquirer disables interrupts when appropriate). The kernel's lockdep tool will check that this rule is followed. 

[ ![\[Boqun Feng\]](https://static.lwn.net/images/2025/boqun-feng-kangrejos-small.png) ](<https://lwn.net/Articles/1041344>)

Every sufficiently complex driver will need an interrupt handler, Feng continued. In order to write drivers in Rust, kernel programmers will need an abstraction to deal with interrupts and irq-safe locks. The existing irq-safe spinlocks in C use either `spin_lock_irq()`/`spin_unlock_irq()` or `spin_lock_irqsave()`/`spin_lock_irqrestore()`, depending on whether the code is expected to run in an interrupt-enabled context or not. The former pair of functions leaves interrupts unconditionally enabled after the lock is released, while the latter pair stores the current interrupt state and restores it afterward. There are also some scope guards that can handle unlocking these spinlocks automatically at the end of a function. 

The first attempt at a Rust API was actually analogous to the kernel's [ scope guards](<https://lwn.net/Articles/934679/>). In Rust, having a special guard type that, when dropped, releases a lock is a common pattern. The problem that Feng ran into was one that the C API struggles with as well: what happens if two locks are dropped in the wrong order? Consider a piece of code that starts out with interrupts enabled, acquires lock A using `spin_lock_irqsave()` (resulting in interrupts being temporarily disabled), and then acquires lock B, again using `spin_lock_irqsave()`. The second call will store "disabled" as the current interrupt state. Releasing lock A and then lock B ends with interrupts being permanently disabled, which ""is not ideal"". 

Feng looked at a few different ways to solve the problem, including using a token type to track whether the code was called in an interruptible context at compile time, but that proved to be too complex. Ultimately, in a [ mailing list discussion](<https://lwn.net/ml/rust-for-linux/20240916213025.477225-1-lyude@redhat.com/>) of the problem in early September, Thomas Gleixner [commented](<https://lwn.net/ml/rust-for-linux/87iktrahld.ffs@tglx/>): 

> Thinking more about this. I think there is a more general problem here. 
> 
> Much of the Rust effort today is trying to emulate the existing way how the C implementations work. 
> 
> I think that's fundamentally wrong because a lot of the programming patterns in the kernel are fundamentally wrong in C as well. They are just proliferated technical debt. 
> 
> What should be done is to look at it from the Rust perspective in the first place: How should this stuff be implemented correctly? 

That sent Feng back to the drawing board; he eventually settled on an approach that tracks the number of nested scopes with interrupts disabled. When the number changes between zero and one, the code can enable or disable interrupts as appropriate, but it no longer matters what order locks are dropped in. That code is written in C, so it can benefit the rest of the kernel as well. 

Andreas Hindborg asked whether all of the existing C users of `spin_lock_irqsave()` would be adapted to Feng's approach, or whether this was a new API. Feng clarified that it was a new API, but that Rust code should only use this one. He said that in the mailing list discussion of the change, scheduling maintainer Peter Zijlstra was [ opposed](<https://lwn.net/ml/rust-for-linux/20241024161759.GF9767@noisy.programming.kicks-ass.net/>) to ending up with _three_ ways to lock and unlock interrupt-aware spinlocks, and ran into problems when he attempted to mechanically convert the existing C users, but Feng was optimistic that Zijlstra would accept a slower transition. 

Joel Fernandes asked how this API interacts with [ `CondVar::wait()`](<https://rust.docs.kernel.org/kernel/sync/struct.CondVar.html#method.wait>), which temporarily releases a spinlock (and reenables interrupts) while waiting for a notification. Feng said that `CondVar::wait()` would save the current depth of interrupt-disabling-nesting, and restore it afterward. Fernandes pointed out that this would only work with one lock held, which Feng agreed with. Waiting on a conditional variable with two locks held is broken anyway ""and lockdep would yell at you"". 

The approach Feng presented nicely enables Rust code to interact with interrupt-aware spinlocks, but there's another small API change that it would enable: Rust could have a separate guard type to enable and disable interrupts (on the local CPU) for non-spinlock reasons. If the running code already has an instance of that type (indicating that it must have disabled interrupts at some point), then it can lock a spinlock without incrementing the count. That would be ""useful in a few obscure cases"", as well as for performance optimization. 

Gary Guo, Benno Lossin, and Feng then launched into a detailed discussion of whether it would be better to let the existing interrupt-aware-spinlock guards serve that purpose, rather than creating a new type. The back and forth quickly escaped me, but it ended with the conclusion that Feng's API is the right path forward, although there may be some ways to avoid the performance overhead of an atomic write in the hot path in more cases in the future. 

At that point, Feng went into the implementation details of the new API. There is a 32-bit, per-CPU field accessed via the [ `preempt_count()`](<https://lwn.net/Articles/831678/>) function that is faster than a normal per-CPU variable; Feng tried seeing if he could steal bits 16 through 23 for the interrupt-disabled nesting count, but that reduced the level of non-maskable interrupt (NMI) nesting that could be tracked from 16 bits to eight bits. Feng wasn't sure why the kernel needs 32,000 levels of NMI nesting, but the maintainer insisted. Since most code only needs to care whether any NMI nesting is happening, not the exact level, Feng and Fernandes ended up reducing the NMI-nesting count to one bit in `preempt_count()`, plus a normal per-CPU variable for the actual depth. 

The existing C users of interrupt-aware spinlocks should, in many cases, be able to just use the new API. Whenever a function has a paired disable and enable, that can safely be replaced with calls to the new `local_interrupt_disable()` and `local_interrupt_enable()` functions. ""But there are some 'creative' unpaired uses."" 

Feng wrote a tool to find unpaired uses, in order to figure out how big a hassle migration would be. In the process, he actually found a latent bug in the kernel where someone had used the unconditional API in a callback when it should have used `local_irq_save()`. He doesn't think that the Rust API should be blocked on rewriting the existing code, though; that can be done more gradually afterward. ""The new API is nicer; so maybe people will voluntarily switch."" 

If Zijlstra and other maintainers aren't happy with that answer, Feng's backup plan is to expose per-CPU variables to Rust, and then write the new API entirely in Rust — leaving the new API unavailable to C users. If most of the existing C users could have been trivially moved to the new, safer API, it would be a shame to miss out on improving the kernel's existing C code. Time will tell whether Feng's proposal is adopted, but, either way, the API for Rust drivers should soon be one step closer to parity with its counterpart for C drivers.
