---
title: Next steps for Rust in the kernel
url: https://lwn.net/Articles/908347/
date: "September 19, 2022"
category: "Development tools-Rust"
author: "By Jonathan Corbet September 19, 2022 Maintainers Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 19, 2022

* * *

[Maintainers Summit](<https://lwn.net/Articles/908320/>)

The Rust for Linux project, which is working to make it possible to write kernel code in the Rust programming language, has been underway for a few years, and there is a growing number of developers who feel that it is time to merge this work into the mainline. At the 2022 Linux Kernel Maintainers Summit, Miguel Ojeda updated the group on the status of the project with the goal of reaching a conclusion on when this merge might happen. The answer that came back was clear enough: Rust in the kernel will be happening soon indeed. 

There was little suspense on that front; Linus Torvalds spoke up at the beginning of the session to say that he plans to accept the Rust patches for the 6.1 kernel (likely to be released in mid-December) unless he hears strong objections. Ojeda indicated that he would like to see that happen and asked how the patches should be routed into the mainline. Torvalds said that he would rather not accept them directly, so it seems likely that Kees Cook will be routing this work upstream. 

[![\[Miguel Ojeda\]](https://static.lwn.net/images/conf/2022/lpc/MiguelOjeda-sm.png)](<https://lwn.net/Articles/908359/>) Dave Airlie said that there are MacBook driver developers who are intent on doing their work in Rust, so there will likely be real Rust drivers heading upstream before too long. Initially, though, Torvalds said that he would like to see a minimal merge just to get the infrastructure into the kernel and allow developers to start playing with it. It should build, but shouldn't do much of anything beyond the "hello, world" stage. That, he said, will be a signal to the world that "it's finally happening". 

Greg Kroah-Hartman asked how subsystem-specific Rust bindings will go upstream; will they go through the Rust tree or via the relevant subsystem maintainers? Ojeda answered that core Rust support will go through the Rust tree, but the rest should go through maintainers. Alexei Starovoitov worried that subsystem maintainers would not be able to refuse Rust patches even if they do not want to see Rust used in their subsystems; James Bottomley added that Rust can be a hard language for longtime C developers to understand, and that it would not be good to force it on maintainers. Torvalds answered that it should be up to the maintainers; there is no need for global rules at this point. 

Paolo Bonzini said that the Rust code implementing abstractions for subsystems is often the most unreadable for developers who are unfamiliar with the language, "but it's stupid code" that is not doing anything complex. Driver-level Rust code is a lot more straightforward. Torvalds repeated that, for now, maintainers will be able to say that they don't want to deal with Rust. Starovoitov countered, though, that BPF will be affected regardless of what he might decide; developers will need to be able to trace Rust code to debug problems. Everybody will need to know Rust eventually, he added. Torvalds replied that he expects that process to take years. 

Cook said that this change will be similar to many of the C language changes that the kernel has gone through. The switch away from variable-length arrays was a similar process, and developers have gotten used to it. Torvalds said that it's closer to the introduction of BPF instead; it's a new language that was initially targeted at specific use cases, but which is now everywhere. 

Ted Ts'o noted that the kernel has to use unstable Rust features, and that creates uncertainty about which version of the language should be used. Perhaps the developers should declare a specific version of the compiler as the one to use for kernel development? That would encourage distributors to package that version, making it more widely available. Thomas Gleixner said that having the blessed compiler available on kernel.org would be good enough, but Torvalds answered that he would rather get compilers from his distributor if possible. Bottomley asked when Rust would become mandatory to build the kernel; the answer was "when the hardware he has requires it". Torvalds said that, if and when that point comes, it will be an indication that Rust is a success for kernel development. 

Gleixner asked about how well specified the Rust language is now; Ojeda answered that it depends on what one is looking for. Rust guarantees backward compatibility for stable features, so those will not break in surprising ways. The kernel, though, is using a number of _unstable_ features; those features are, unsurprisingly, unstable. Work is being done to stabilize those features so that the kernel will be able to count on them going forward. 

There is currently an ongoing effort to write a specification for Rust for safety-critical systems that will lead to a standard-like document. At the moment, though, Ojeda said, the developers of the GCC-based gccrs Rust compiler are finding the current documentation to be vague at times. Often, behavior is specified as "whatever the rustc compiler does". That is "not good", he said, but there is a path forward. 

Gleixner also inquired into the tools that are generating the Rust bindings and, specifically, whether there is automation to ensure that the Rust and C versions of data structures match each other. Those tools do exist, Ojeda said, but they do not yet automatically convert all types successfully. That can be fixed. 

Finally, Gleixner admonished the Rust developers to not change the semantics of any C locking primitives; it's worth noting that they have shown no inclination to do that so far. Ts'o added that Rust's locking abstractions should be made to work with the lockdep locking checker from the beginning. Chris Mason interjected that, if lockdep is needed for Rust code, that will be another sign that the language has succeeded and it will be time to "do a victory dance". 

It has often been said that the merging of Rust into the kernel tree will be done on an experimental basis; if it doesn't work out, it can be removed again. Ojeda said that the developers working on Rust for Linux would like to know how long the trial period is likely to be. He did not really get an answer from the group, though. 

Instead, Bottomley suggested that, rather than bringing in Rust, it might be better to just move more Rust-like features into C. Ojeda said that he has actually been working with the C language committee to push for that to happen, but any such change will take a long time if it happens at all. Christoph Hellwig said that this sort of change will have to happen anyway unless the plan is to rewrite the whole kernel in Rust; he was not pleased at the idea of rewriting working code in a new language. Perhaps the [sparse](<https://lwn.net/Articles/689907/>) static analyzer could be enhanced to do more Rust-like checking, he said. Ojeda answered that the result of such efforts would be like having Rust — but much later. 

Hellwig continued that the adoption of Rust-like features could be done incrementally over time. It would be "strictly worse than starting in Rust", but the kernel community has a massive code base to manage. There needs to be a way to get the benefits of a Rust-like language into all of that C code, he said. Cook said he's been pushing compiler developers to create safer C dialects as well. 

Ts'o brought the session to a conclusion by noting that language design is a long-term research project; perhaps the group should focus on policy issues for the next year instead. Torvalds said that he would like to see the groups running continuous-integration testing services to incorporate Rust testing — something that is already happening. Laurent Pinchart said that the Rust developers need to be ready to provide support to the kernel community in the early days; developers will pick things up quickly and be able to help each other after a while. Torvalds added that Rust isn't that terrible in the end; "it's not Perl". 

When asked about documentation, Ojeda said that the Rust developers are trying to improve on the documentation that has been done on the C side. The Rust documentation mechanism makes it easy to ensure that examples are actually tested, for example. They are adhering to rules on how unsafe blocks should be explained. 

As time ran out, Matthew Wilcox asked whether kernel developers should be writing idiomatic Rust code, or whether they will be writing "C in Rust". Ojeda answered that code might be more C-like toward the beginning; adoption of more advanced features (such as async) might take longer. Gleixner asked what could be done to prevent developers from using unstable features (once the features used by the kernel are stabilized); the answer was to specify the version of the compiler to be used with kernel development.
