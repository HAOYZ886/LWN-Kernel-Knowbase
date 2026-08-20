---
title: Using LKMM atomics in Rust
url: https://lwn.net/Articles/993785/
date: "October 16, 2024"
category: "Development tools-Rust"
author: "By Daroc Alden October 16, 2024 Kangrejos 2024"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Daroc Alden**  
October 16, 2024

* * *

[Kangrejos 2024](<https://lwn.net/Articles/990496/>)

Rust, like C, has its own memory model describing how concurrent access to the same data by multiple threads can behave. The Linux kernel, however, has its own ideas. The [ Linux kernel memory model](<https://lwn.net/Articles/718628/>) (LKMM) is subtly different from both the standard C memory model and Rust's model. At Kangrejos, Boqun Feng gave a presentation about the need to reconcile the memory models used by Rust and the kernel, including a few potential avenues for doing so. While no consensus was reached, it is an area of active discussion. 

The problem, Feng explained, is that the LKMM makes guarantees that the Rust memory model does not. Since the compiler doesn't know about those guarantees, it can (potentially) make optimizations that break them. The only saving grace is the ABI between C and Rust code, which should have certain guarantees that both sides are aware of. However, in practice, many architectures don't specify any guarantees about atomic operations or interactions between threads as part of their ABI. Having an ABI that listed the relevant guarantees wouldn't be a complete solution in any case — cross-language link-time-optimization (LTO) could still cause problems, Feng said. 

[ ![\[Boqun Feng\]](https://static.lwn.net/images/2024/boqun-feng-small.png) ](<https://lwn.net/Articles/994272>)

We need the memory models to admit the existence of each other, he stated. It might seem as though pure-Rust code would not need to care about the LKMM, but even that is not really true. The LKMM guarantees that if one thread stores to a variable and then wakes another thread, the second thread will see that store. Rust doesn't know that, and so the compiler could theoretically reorder the store after the call to wake another thread. So any Rust code is impacted by the things that the LKMM requires. 

Andreas Hindborg asked whether there were really no situations where a kernel driver written completely in Rust could get away with not using the LKMM. Feng gave an example of how even a simple multithreaded atomic counter ends up involving the LKMM. Paul McKenney summed the problem up like this: there are plenty of existing boundaries where someone might suggest making a distinction and saying that one memory model applies on one side and one on the other side — function calls, for example — but we don't do that. Ordering has to be a global property, McKenney said, or things will get complicated for tool writers. 

Benno Lossin questioned why Hindborg would want to use Rust's memory model (specifically, the atomic operations that it makes available) for isolated sections of code, if the Rust-for-Linux project is going to need versions that match the LKMM anyway. Unlike any future attempt at producing a LKMM-compatible atomics API for Rust, the existing Rust atomics are available now, Hindborg said, and he doesn't want to be slowed down. McKenney suggested a phased approach — target eventual exact compatibility, but for now, strategically placed full barriers could be sufficient, even if they have higher overhead. 

Alice Ryhl suggested adding new types that are intended to eventually match the semantics of the LKMM, implement them with Rust atomics internally, and then later redesign them. Lossin disagreed, saying that the API design was the hard part, and that it made more sense to use Rust atomics for now, and fix it once the API did actually exist. Gary Guo suggested an entirely different approach: checking that the compiled machine code respects the LKMM, regardless of what the source languages are. If we can have LKMM atomics in Rust, we should just use them, he said. 

Feng's presentation came to the same conclusion: that the Rust-for-Linux project should implement LKMM-compatible atomics and other related abstractions in Rust, and use only those. To explain what that implies for the people who may be less familiar with the LKMM, Feng highlighted a few specific differences. For one thing, all atomic variables are automatically assumed to be volatile as well — so the compiler cannot invent extraneous loads or redundant stores to them. For another, there are different atomic orderings available, including ""fully ordered"", which acts as a full barrier for any other atomic operations. Failed compare-and-exchange operations count as relaxed memory operations (as opposed to having two different versions, one of which is always relaxed and one of which isn't). Finally, the LKMM adds address, data, and control dependencies that can influence ordering. Some of those are particularly subtle — an if statement with a condition that reads an atomic variable only orders subsequent atomic writes, not subsequent atomic reads, for example. 

It is tempting to assume that, since Rust code compiles to the same LLVM intermediate representation as C code, the compiler should be able to respect the rules of the LKMM in the same way. Unfortunately, C compilers actually already cause problems for C code trying to follow the LKMM. Feng gave the example of code trying to take advantage of the control dependencies he mentioned. Imagine an if statement that reads from an atomic variable, and then writes to a different variable in both branches of the if statement before going on to do two different things. The compiler can and does [ hoist the identical writes out of the if statement](<https://lwn.net/Articles/860037/>) — something that would not cause a problem for ordinary code, but that can change the order of atomic operations and potentially break the guarantees that the programmer was relying on. In the kernel, this is the reason for the `volatile_if()` and `ctrl_dep()` macros, which generate appropriate compiler barriers to prevent that from happening. 

Guo asked whether the Rust [ `black_box()`](<https://doc.rust-lang.org/std/hint/fn.black_box.html>) function could serve a similar purpose, and Feng agreed that it could. McKenney was skeptical that it would help with control dependencies in particular — and a quick test by Guo confirmed that it does not. But there are other potential solutions based on Rust macros. 

In any case, the solution will certainly involve paying more attention to how atomics are used in Rust code. And while it is tempting to use simpler implementations, this is the kernel — so there's no real way to avoid caring about performance and architectural details, Feng concluded. There is some hope for creating a generic API that the Rust-for-Linux project could implement, however. Rust may soon have [ generic atomics](<https://github.com/rust-lang/rust/issues/130539>) in the form of an `Atomic` type that unifies all of the existing atomic APIs. The kernel developers could theoretically implement the same API, but based on LKMM atomics. 

As the session was wrapping up, Ryhl said that she didn't care whether they end up implementing any particular API — she just thinks that they should focus on doing something simple first. The session came to a close before the attendees could agree on what that might be, but, either way, the memory model consistency concerns Feng wanted to address are definitely being actively considered.
