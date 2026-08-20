---
title: "READ_ONCE(), WRITE_ONCE(), but not for Rust"
url: https://lwn.net/Articles/1053142/
date: "January 9, 2026"
category: "Development tools-Rust; Lockless algorithms"
author: "By Jonathan Corbet January 9, 2026"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 9, 2026

The `READ_ONCE()` and `WRITE_ONCE()` macros are heavily used within the kernel; there are nearly 8,000 call sites for `READ_ONCE()`. They are key to the implementation of many [lockless algorithms](<https://lwn.net/Articles/844224/>) and can be necessary for some types of device-memory access. So one might think that, as the amount of Rust code in the kernel increases, there would be a place for Rust versions of these macros as well. The truth of the matter, though, is that the Rust community seems to want to take a different approach to concurrent data access. 

An understanding of `READ_ONCE()` and `WRITE_ONCE()` is important for kernel developers who will be dealing with any sort of concurrent access to data. So, naturally, they are almost entirely absent from the kernel's documentation. A description of sorts can be found at the top of [`include/asm-generic/rwonce.h`](<https://elixir.bootlin.com/linux/v6.18.3/source/include/asm-generic/rwonce.h>): 

> Prevent the compiler from merging or refetching reads or writes. The compiler is also forbidden from reordering successive instances of READ_ONCE and WRITE_ONCE, but only when the compiler is aware of some particular ordering. One way to make the compiler aware of ordering is to put the two invocations of READ_ONCE or WRITE_ONCE in different C statements. 

In other words, a `READ_ONCE()` call will force the compiler to read from the indicated location exactly one time, with no optimization tricks that would cause the read to be either elided or repeated; `WRITE_ONCE()` will force a write under those terms. They will also ensure that the access is atomic; if one task reads a location with `READ_ONCE()` while another is writing that location, the read will return the value as it existed either before or after the write, but not some random combination of the two. These macros, other than as described above, impose no ordering constraints on the compiler or the CPU, making them different from macros like `smp_load_acquire()`, which have stronger ordering requirements. 

The `READ_ONCE()` and `WRITE_ONCE()` macros were [added for the 3.18 release](<https://git.kernel.org/linus/230fa253df635>) in 2014. `WRITE_ONCE()` was initially called `ASSIGN_ONCE()`, but that name was [changed](<https://git.kernel.org/linus/43239cbe79fc3>) during the 3.19 development cycle. 

On the last day of 2025, Alice Ryhl posted [a patch series](<https://lwn.net/ml/all/20251231-rwonce-v1-0-702a10b85278@google.com>) adding implementations of `READ_ONCE()` and `WRITE_ONCE()` for Rust. There are places in the code, she said, where volatile reads could be replaced with these calls, once they were available; among other changes, the series [changed access to the `struct file` `f_flags` field](<https://lwn.net/ml/all/20251231-rwonce-v1-5-702a10b85278@google.com>) to use `READ_ONCE()`. The [implementation](<https://lwn.net/ml/all/20251231-rwonce-v1-2-702a10b85278@google.com>) of these macros involves a bunch of Rust macro magic, but in the end they come down to calls to the Rust [`read_volatile()`](<https://doc.rust-lang.org/std/ptr/fn.read_volatile.html>) and [`write_volatile()`](<https://doc.rust-lang.org/stable/std/ptr/fn.write_volatile.html>) functions. 

Some of the other kernel Rust developers objected to this change, though. Gary Guo [said](<https://lwn.net/ml/all/20251231151216.23446b64.gary@garyguo.net>) that he would rather not expose `READ_ONCE()` and `WRITE_ONCE()` and suggested using relaxed operations from ~~[the Rust` Atomic` crate](<https://doc.rust-lang.org/std/sync/atomic/>)~~ the kernel's [`Atomic`](<https://rust.docs.kernel.org/next/kernel/sync/atomic/struct.Atomic.html>) module instead. Boqun Feng [expanded on](<https://lwn.net/ml/all/aVXKP8vQ6uAxtazT@tardis-2.local>) the objection: 

> The problem of READ_ONCE() and WRITE_ONCE() is that the semantics is complicated. Sometimes they are used for atomicity, sometimes they are used for preventing data race. So yes, we are using LKMM [the Linux kernel memory model] in Rust as well, but whenever possible, we need to clarify the intention of the API, using Atomic::from_ptr().load(Relaxed) helps on that front. 
> 
> IMO, READ_ONCE()/WRITE_ONCE() is like a "band aid" solution to a few problems, having it would prevent us from developing a more clear view for concurrent programming. 

In other words, using the `Atomic` crate allows developers to specify more precisely which guarantees an operation needs, making the expectations (and requirements) of the code more clear. This point of view would appear to have won out, and Ryhl has stopped pushing for this addition to the kernel's Rust code — for now, at least. 

There are a couple of interesting implications from this outcome, should it hold. The first of those is that, as Rust code reaches more deeply into the core kernel, its code for concurrent access to shared data will look significantly different from the equivalent C code, even though the code on both sides may be working with the same data. Understanding lockless data access is challenging enough when dealing with one API; developers may now have to understand two APIs, which will not make the task easier. 

Meanwhile, this discussion is drawing some attention to code on the C side as well. As Feng [pointed out](<https://lwn.net/ml/all/aV0JkZdrZn97-d7d@tardis-2.local>), there is still C code in the kernel that assumes a plain write will be atomic in many situations, even though the C standard explicitly says otherwise. Peter Zijlstra [answered](<https://lwn.net/ml/all/20260106145622.GB3707837@noisy.programming.kicks-ass.net>) that all such code should be updated to use `WRITE_ONCE()` properly. Simply finding that code may be a challenge (though [KCSAN](<https://docs.kernel.org/dev-tools/kcsan.html>) can help); updating it all may take a while. The conversation also [identified](<https://lwn.net/ml/all/87ikdej4s1.fsf@t14s.mail-host-address-is-not-set>) a place in the (C) high-resolution-timer code that is missing a needed `READ_ONCE()` call. This is another example of the Rust work leading to improvements in the C code. 

In past discussions on the design of Rust abstractions, there has been resistance to the creation of Rust interfaces that look substantially different from their C counterparts; see [this 2024 article](<https://lwn.net/Articles/958072/>), for example. If the Rust developers come up with a better design for an interface, the thinking went, the C side should be improved to match this new design. If one accepts the idea that the Rust approach to `READ_ONCE()` and `WRITE_ONCE()` is better than the original, then one might conclude that a similar process should be followed here. Changing thousands of low-level concurrency primitives to specify more precise semantics would not be a task for the faint of heart, though. This may end up being a case where code in the two languages just does things differently.
