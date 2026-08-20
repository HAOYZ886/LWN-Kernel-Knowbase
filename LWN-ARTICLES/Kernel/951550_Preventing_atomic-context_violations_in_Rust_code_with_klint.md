---
title: "Preventing atomic-context violations in Rust code with klint"
url: https://lwn.net/Articles/951550/
date: "November 17, 2023"
category: "Development tools-Rust"
author: "By Jonathan Corbet November 17, 2023 LPC"
---

By **Jonathan Corbet**  
November 17, 2023

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2023-Linux_Plumbers_Conference>)

One of the core constraints when programming in the kernel is the need to avoid sleeping when running in atomic context. For the most part, the responsibility for adherence to this rule is placed on the developer's shoulders; Rust developers, though, want the compiler to ensure that code is safe whenever possible. At the [2023 Linux Plumbers Conference](<https://lpc.events/>), Gary Guo presented (via a remote link) the klint tool, which can find and flag many atomic-context violations before they turn into user-affecting bugs. 

Rust is built on the idea that safe Rust code, as verified by the compiler, cannot cause undefined behavior. This behavior comes in a lot of forms, including dereferencing dangling or null pointers, buffer overruns, data races, or violations of the aliasing rules; code that is "safe" will not do those things. The Rust-for-Linux project is trying to create an environment where much kernel functionality can be implemented with safe code. On the other hand, some surprising behavior, including memory leaks, deadlocks, panics, and aborts, is considered "safe". This behavior is defined, thus "safe" (though still, obviously, bad). 

Atomic context in the kernel raises some interesting safety questions. If code, for example, executes a sequence like: 

```
spin_lock(&lock);
        /* ... */
        mutex_lock(&mutex);   /* can schedule */
        /* ... */
        spin_unlock(&lock);
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

the result could be a deadlock if another thread attempts to take the same spinlock on the same CPU. That is "safe" (but "bad") code. But what about code like the following? 

```
rcu_read_lock();
        schedule();
        rcu_read_unlock();
```

In this case, the safety of this code, even in the Rust sense, is not so clear. RCU assumes that there will be no context switches in code that is running within an RCU critical section; calling into the scheduler breaks that assumption. In this case, the atomic-context violation can indeed be a safety issue, creating use-after-free bugs, data races and worse. This is "fine" for C code, where the distinction between "safety" and "correctness" is not so well defined. Rust developers, though, try to live by different rules; consequently, they cannot design a safe API that allows sleeping in atomic context. 

[![\[Gary Guo\]](https://static.lwn.net/images/conf/2023/lpc/GaryGuo-sm.png)](<https://lwn.net/Articles/951776/>) Avoiding that situation is not easy, though. One possible solution would be to make all blocking operations unsafe. That, Guo acknowledged, is likely to be widely seen as a bad idea. Another approach is token types, which are commonly used in Rust to represent capabilities; functions that might sleep can require a token asserting the right to do so. That leads to complex and unwieldy APIs, though. It is possible to do runtime checking, using the preemption count maintained in some kernel configurations now. That adds runtime overhead, though, and the preempt count is not available in all kernel configurations. 

The last option would be to simply ignore the problem and trust developers to get things right, perhaps using the kernel's lockdep locking checker to find some problems on development systems. That approach, though, is unsound and not the Rust way of doing things. 

The root of the problem, Guo said, is the need to optimize three objectives (soundness, an ergonomic API, and minimal run-time overhead) in a "choose any two" situation. Token types optimize soundness and overhead at the expense of an ergonomic API, for example, while run-time checking improves the API but sacrifices the goal of avoiding run-time overhead. Solutions that optimize all three quantities are hard to come by; the kernel's needs simply do not fit nicely into the Rust safety model. 

The answer, Guo said, is to adapt the Rust compiler to this use case; that has been done in the form of a tool called "klint", which will verify at compile time the absence of atomic-context violations to the maximum extent possible. For the cases that cannot be verified, an escape hatch, in the form of a run-time check or use of `unsafe`, will be provided to developers so that their code can be built. 

This tool was built with a number of goals in mind. It should be easy to explain and understand, of course, and provide useful diagnostics. There needs to be an escape hatch so that it does not get in the way of getting real work done. Its defaults, he said, should be sane, and there should be little need for additional annotations in the kernel. Finally, the tool needs to be fast so that it can be run every time the code is built. 

Klint gives every function two properties, the first of which is an "adjustment" describing the change it makes (if any) to the [preemption count](<https://lwn.net/Articles/831678/>) (which, when non-zero, indicates that the current thread cannot be preempted). The second is the expected value of the preemption count when the call is made; this value can be a range. The klint tool tracks the possible state of the preemption count at each location, looking for situations where a function's expected preemption count is violated. 

Thus, for example, `rcu_read_lock()` increments the preemption count by one, and can be called with any value. In Rust code, that would be annotated as: 

```
#[klint::preempt_count(adjust = 1, expect = 0.., unchecked)]
        pub fn rcu_read_lock() -> RcuReadGuard { /* ... */ }
```

As klint passes over the code, it tracks the possible values of the preemption count and flags an error if an expected condition is not met. For example, `schedule()` would be annotated as expecting the preemption count to be zero; if klint sees a call to `schedule()` after a call to `rcu_read_lock()` it will complain — unless, of course, there is a call to `rcu_read_unlock()` that happens first. 

The compiler's type inference makes explicit annotation unnecessary much of the time. There are exceptions, naturally, including at the foreign-function interface boundary, with recursive functions, and with indirect function calls. Other limitations exist as well; there is, for example, no way to annotate functions like `spin_trylock()`, where the effect on the preemption count is not known in advance. Perhaps, in the future, that shortcoming could be addressed by adding some sort of `match` expression to the annotations, he said. 

Data-dependent acquisition, where, for example, a function only takes a lock if a boolean parameter instructs it to, is also not handled by klint at this point. Finally, there are cases where the compiler injects code into a function that confuses klint, leading to incorrect reports. This problem is currently blocking the wider use of klint, and is thus urgent to solve. Meanwhile, he said, klint imposes a negligible compile-time overhead. 

Guo concluded by saying that klint is [available on GitHub](<https://github.com/rust-for-linux/klint>) for folks who want to play with it. More information can also be found in [the slides from the talk](<https://lpc.events/event/17/contributions/1431/attachments/1336/2679/klint.pdf>). 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting our travel to this event.]
