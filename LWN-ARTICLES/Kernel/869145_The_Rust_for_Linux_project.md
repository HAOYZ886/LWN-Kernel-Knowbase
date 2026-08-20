---
title: The Rust for Linux project
url: https://lwn.net/Articles/869145/
date: "September 16, 2021"
category: "Development tools-Rust"
author: "By Jonathan Corbet September 16, 2021 Kangrejos"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 16, 2021

* * *

[Kangrejos](<https://lwn.net/Archives/ConferenceByYear/#2021-Kangrejos>)

The first ever Rust for Linux conference, known as [Kangrejos](<https://kangrejos.com/>), got underway on September 13\. Organizer Miguel Ojeda used the opening session to give an overview of why there is interest in using Rust in the kernel, where the challenges are, and what the current status is. The talk and following discussion provided a good overview of what is driving this initiative and where some of the sticking points might be. 

#### Why Rust?

This year marks the 30th anniversary of the Linux kernel, he began, but it is also the 30th anniversary of the first ISO standard for the C programming language. C has a lot of history and a lot of baggage, but it is still relevant and interesting. Ojeda said that he is working with the C committee to improve the language, but that process will take a long time — if it ever reaches fruition. 

C is good for kernel development for a number of reasons. It is fast, facilitates low-level programming, is simple, and fits the problem domain well. But there is one little problem, known generally as "undefined behavior". The advantage of Rust is that it eliminates undefined behavior, at least in "safe" code. Indeed, the lack of undefined behavior is what Rust developers [![\[Miguel Ojeda\]](https://static.lwn.net/images/conf/2021/rust/MiguelOjeda1-sm.png)](<https://lwn.net/Articles/869240/>) mean when they say "safe"; it has no relation to similar terms like "safety-critical". An `abort()` call in C is safe by this definition, even if the end result kills somebody. Ojeda has proposed a "`[[safe]]`" attribute for C to mark functions that are written to avoid all undefined behavior (and have the compiler enforce that), but that proposal has not made much progress. 

So what is not "safe" by Rust standards? The list of not-safe behaviors include using pointers after they are freed, dereferencing NULL pointers, freeing memory twice, using the contents of uninitialized memory, out-of-bounds memory accesses, data races, and more. None of these things will happen, he asserted, in safe sections of Rust code; that is why he wants to see Rust code in the kernel. About 70% of published C vulnerabilities result from undefined behavior, he said, and the rate for vulnerabilities in the Android media and Bluetooth components is closer to 90%. Rust can help make those problems go away. Rust offers a number of other advantages as well, including stricter types, modules, pattern-matching primitives, lifetimes, an extensive set of development tools, and much more. 

What is the downside of using Rust in the kernel? It is not possible to model everything, he said, so it is still necessary to use some unsafe code. The Rust language is much more complex than C, and its extra run-time checks can hurt performance. And, of course, it is another language for kernel developers and maintainers to learn, which is also a significant cost. It's a one-time cost that, he said, pays off, but it's still a cost. The Rust for Linux developers are happy to help developers get past this initial learning obstacle. 

So how does Rust compare to C as a systems-programming language? Like C, it is fast. It's somewhat low-level like C but, depending on the code, it might be less easy for developers to predict what the resulting assembly code will look like. Rust is definitely _not_ simple like C is. He believes that it fits the domain as well as C does, but others may differ. 

#### Rust support in the kernel

There are five supported architectures for Rust so far: armv6, arm64, powerpc, riscv, and x86-64. Work to this point has not been aimed at supporting every possible architecture; that is a lot of low-level work that doesn't necessarily demonstrate anything new. Instead, the intent is to show that supporting a variety of architectures is possible. There are currently three projects working on compiling Rust code for the kernel. Two of them use the "official" rustc front-end, then using LLVM or GCC for code generation. The rustc/LLVM combination is the leading compiler at this point; Ojeda did not say much about the rustc/GCC pairing. There is also a project working on [a native Rust compiler for GCC (gccrs)](<https://github.com/Rust-GCC/gccrs>); that is expected to be ready in a year or two. 

The Rust for Linux work is split between the Rust and kernel trees. The Rust side has two crates called `core` (low-level functionality) and `alloc` (data structures and such). The `alloc` crate is part of the submission for the kernel side as well; that might eventually not be necessary. These crates are considered to be more a part of the standard Rust library than the kernel. 

On the kernel side there are the `kernel` and `builtins` crates. The `kernel` crate abstracts the interface to the rest of the kernel. The `bindgen` tool is used to generate the bindings that allow calling kernel functions from Rust; there is currently no provision for calling back into Rust from C. One problem that still needs a complete solution, he said, is keeping the C and Rust sides synchronized. The intent is for Rust to be a first-class citizen in the kernel; if a developer makes a change that breaks Rust code, they should fix it, just as they would with the kernel's C code. Just how that will work is not yet fully clear, though. 

Where things get interesting is the driver author's point of view. It would be possible to create Rust bindings for all of the kernel functions that one normally calls from a driver and have it work, but that is not what the Rust-on-Linux developers are trying to do. Instead, they are populating the `kernel` crate with a set of subsystem abstractions and interfaces that will, it is hoped, make it possible to write drivers with no unsafe code at all, he said. 

#### Discussion

There were some concerns raised at this point about the need for maintainers to learn Rust. Ojeda acknowledged that this will be necessary; in the end, maintainers need to be able to take responsibility for their subsystems, even if the Rust developers help them initially. Laurent Pinchart said that help could be needed for a long time; he does not have the time to stop everything and learn Rust anytime soon, so he will be hard put to take responsibility for any Rust code in his subsystem. 

Ojeda answered that the Rust developers are aware of this; they have been trying to pick subsystems to start that have maintainers with the time and interest to take up this challenge. Hopefully that will result in the creation of enough examples and experience to help bring in other [![\[Discussion\]](https://static.lwn.net/images/conf/2021/rust/day1-discussion-sm.png)](<https://lwn.net/Articles/869241/>) subsystems later. He reiterated, though, that maintainers need to be on board; if Rust isn't a first-class citizen in the kernel, this experiment is not going to work. Half in jest (perhaps), he added that maintainers who jump on the Rust train sooner are likely to get more help from the Rust developers than those who come later. 

Ojeda said that, while subsystem maintainers may have some difficult work to do, life will be easier for driver writers, who only need to work with the (safe) interfaces provided to them. Pinchart objected that driver writers often find that they need to make core-subsystem changes to support their devices properly, so life isn't quite that simple. If the bar for driver authors is raised to include being fluent in Rust, that will be a real impediment for some, he said. 

Paolo Bonzini worried that Rust code could inhibit changes to internal kernel APIs. A developer may find that Rust code is the only remaining user of a given API, but may lack the desire (or the ability) to fix that code, so the old API will remain. He asked: how many wrappers for core-kernel APIs do the Rust developers plan to add? If, for example, only some `file_operations` members are used by Rust code, will there still be bindings for all of them? Adding more bindings than necessary may make it harder to change those operations in the future. Ojeda replied that, for something as core as `file_operations`, they might just add them all, since they will be used eventually. In other cases, only the interfaces actually used by Rust code will be implemented. 

The conversation wandered into the toolchain; Ojeda said that the Rust for Linux developers are currently only supporting one version of the Rust compiler with any given kernel version. Others may work, but there is no guarantee. Eventually it will be possible to build the kernel using only stable Rust features, at which point it will be possible to support more compiler versions. He added that concerns about the stability of the Rust language in general have been raised, but he does not think that is a problem. The language is stable now, he said, and becoming more stable quickly. Kernel code written now, even code using unstable features, will "mostly" work going forward. 

Philip Herron, who is working on gccrs, asked: what areas in particular would the Rust for Linux developers like to see written in Rust? Ojeda suggested filesystems and other subsystems that contain a lot of state would be good candidates. Herron then asked about whether there were any plans to pick a specific subset of the Rust language to use in the kernel. He asked specifically about [const generics](<https://blog.rust-lang.org/2021/02/26/const-generics-mvp-beta.html>) which, he said, would be a good feature to use in the kernel context. But will there come a time when the use of language features must be limited to reduce the burden on maintainers? Ojeda said that some of the more esoteric features might be useful in the core code, but he would rather see drivers using a smaller subset of the language and avoid things like "crazy metaprogramming" and such. 

At this point, the session ran out of time and the participants headed off for a much-needed coffee break; further discussion was deferred to the following two days. More than 30 developers attended the session, indicating that there is a fair amount of interest in the idea of writing kernel code in Rust, even if that capability remains out of the mainline kernel for now. Rust in the kernel is not an easy sell, but if it fails, it will not be due to a lack of trying.
