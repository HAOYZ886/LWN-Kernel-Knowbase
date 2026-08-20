---
title: Committing to Rust for kernel code
url: https://lwn.net/Articles/952029/
date: "November 22, 2023"
category: "Development tools-Rust"
author: "By Jonathan Corbet November 22, 2023 Maintainers Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 22, 2023

* * *

[Maintainers Summit](<https://lwn.net/Articles/951847/>)

Rust has been a prominent topic at the Kernel Maintainers Summit for the last couple of years, and the 2023 meeting continued that tradition. As Rust-for-Linux developer Miguel Ojeda noted at the beginning of the session dedicated to the topic, the level of interest in using Rust for kernel development has increased significantly over the last year. But Rust was explicitly added to Linux as an experiment; is the kernel community now ready to say that the experiment has succeeded? 

The Rust-for-Linux project has added a full-time engineer in the last year, Ojeda said, and a student developer as well. Various companies have joined in to support this work. There is also work underway to get the [Coccinelle](<https://lwn.net/Articles/698724/>) tool working with Rust code. A priority at the moment is bringing in more reviewers for the code that is being posted. 

On the toolchain front, work on [gccrs](<https://github.com/Rust-GCC/gccrs>), the GCC-based Rust compiler, has slowed significantly. The [GCC code generator for `rustc`](<https://github.com/rust-lang/rustc_codegen_gcc>) is showing better progress; it can compile kernel code now and has been merged into the compiler. This GCC-based backend will enable the expansion of Rust support to architectures that are not supported by the LLVM-based `rustc`. Meanwhile, the Rust project itself is increasing its involvement in this work; this is good, since the kernel has some unique requirements and will need guarantees that language changes won't break kernel code in the future. 

[![\[Miguel Ojeda\]](https://static.lwn.net/images/conf/2023/lpc/MiguelOjeda-sm.png)](<https://lwn.net/Articles/952032/>) Within the kernel, work is proceeding in a number of subsystems. The Rust implementation of Android's binder is working well and its performance is on a par with the C implementation. The amount of unsafe code that was needed to get there was pleasingly small. Filesystem bindings are the subject of work by Wedson Almeida Filho, who is targeting read-only support for now. The object there is to make it possible to implement a filesystem in 100% safe Rust. 

In general, he is finding an increasing number of maintainers who are open to the idea of using Rust. That leads to an issue the Rust developers have run up against, though. It would be good to have some reference drivers in the kernel as an example of how drivers can be written and to make it possible to compare Rust and C drivers. The best way to do that often seems to be to merge a Rust driver that duplicates the functionality of an existing C driver — but that sort of duplicate functionality is not welcomed by maintainers. Perhaps, he said, it would be good to allow a few duplicate drivers that are not meant for actual use, but only as examples for other developers to use. 

There are some other challenges; upstreaming the block-layer abstractions has run into some resistance. Virtual filesystem layer maintainer Christian Brauner said that he is not opposed to merging those abstractions, but he would rather not do that and see filesystems built on it right away. He would prefer to see an implementation of something relatively simple, along the lines of the binder driver, to show that things work as expected. 

#### A driver soon?

Dave Airlie, the maintainer of the DRM (graphics) subsystem, said that, if he has his way, there will be a Rust DRM driver merged within the next couple of releases. Christoph Hellwig shot back that Airlie was willing to "make everybody's life hell" so that he could play with his favorite toy. Merging Rust, Hellwig said, would force others to deal with a second language, new toolchains, and "wrappers with weird semantics". Dan Williams said that the current situation "is what success looks like", and that the kernel community was already committed to Rust. 

Airlie continued that a lot of the Rust work is currently blocked in a sort of chicken-and-egg problem. Abstractions cannot be merged until there is a user for them, but the code needing those abstractions is blocked waiting for code to land in multiple subsystems. As a result, developers working on Rust are dragging around large stacks of patches that they need to get their code to work. Breaking that roadblock will require letting in some abstractions without immediate users. Ojeda agreed that this problem has been slowing progress, but said he has tried not to put pressure on maintainers to merge code quickly. In the case of networking, ironically, the Rust developers [had to ask the networking maintainers to slow down](<https://lwn.net/Articles/949270/>) merging Rust code. 

The conversation took several directions from there. Greg Kroah-Hartman said that merging the binder driver would be a good next step; it is self-contained, has a single user that is committed to its maintenance, and doesn't touch the rest of the kernel. Kees Cook disputed the description of Rust as a "toy", saying that there is a lot of pressure to not use C for new code; Hellwig responded that the developers would have to rewrite everything in Rust, otherwise the resulting dual-language code base would be worse than what exists now. 

Dave Chinner worried that maintainers lack the expertise to properly review the abstractions that are being merged. Airlie replied that maintainers merge a lot of C APIs now without really understanding how they work. A lot of mistakes have been made in the process, but "we're still here". When things turn out to be broken, they can be fixed, and that will happen more quickly if the code goes upstream. 

Ted Ts'o expressed concern about the burden that adding Rust code will place on maintainers. The Rust developers are setting higher standards than have been set in the past, he said. Getting good abstractions merged is one thing, but who is responsible for reviewing drivers, and how will tree-wide changes be handled? The Rust effort, he said, is getting to a point where it is impacting a growing part of the community. 

Williams pointed out that [the previous session](<https://lwn.net/Articles/951846/>) had discussed how hard it is to get kernel subsystems to move to new APIs; now, he said, there is talk of moving to a whole new language. Hellwig said that the real problem is that the Rust bindings tend to work differently than the C APIs they provide abstractions for; the new APIs may well be better, but they are still completely new APIs. What should be done, he said, is to first fix the C APIs so that they are directly usable by Rust code. He proposed that, for each subsystem that is considering bringing in Rust code, a year or two should first be spent on cleaning up its APIs along those lines. Ojeda said that this kind of API improvement has already happened in some subsystems. 

Linus Torvalds said that he was seeing a divide between the filesystem and driver maintainers. Developers on the filesystem side tend to be more conservative, while the driver world "is the wild west". Driver authors tend not to understand concurrency, he said, and a lot of the code there is broken and unfixable. So it is unsurprising that there is interest in bringing in a language that better supports the writing of correct and safe code. 

Brauner said that Rust can help with a lot of problems, since the compiler can keep a lot of bugs from making it into the kernel. But he worried about whether there would be maintainer and development support for it a few years from now. Airlie again mentioned developers with out-of-tree code needed by Rust code; Cook answered that the people shepherding that code _are_ maintainers, and that bringing it in would bring the maintainers with it. Airlie added that those maintainers are the sort of younger developers that the kernel community would like to attract. 

Chinner said that he would like to see a reimplementation of the ext2 filesystem in Rust. It is a complete filesystem that makes wide use of the kernel's APIs, but it is still small enough to read and understand. If the Rust APIs can support an ext2 implementation, they will be enough to implement others as well. Meanwhile, the ext2 implementation would be good reference for maintainers, who could compare it to the C version. 

#### Confidence

Ts'o asked when the community would feel enough confidence that it could have modules where the only implementation is in Rust. Binder could be a good start, he said, perhaps followed by a driver that sees wider use. Airlie said that he is considering a virtual graphics driver that reimplements an existing C driver. There is also the driver for Apple M1 GPUs. He is feeling a fair amount of pressure to get it upstream and is wondering if there is any reason why he should keep it out. After that, he would love to see a rewrite of the Nouveau driver for NVIDIA GPUs. 

Arnd Bergmann said those drivers could be OK, but that it will be quite a bit longer before something like a keyboard driver could be merged; the toolchain just isn't ready, he said, for a driver what would be widely used. That led to a question about the frequent version upgrades being seen in the kernel, which moved to Rust 1.73.0 for 6.7. That upgrade process will eventually stop and a minimum Rust version will be set once the important features that the kernel depends on have stabilized. He said that he has been working to get the kernel code into the Rust continuous-integration tests to help ensure that it continues working as the compiler and language evolve. 

Bergmann said that he didn't plan to look seriously at the language until it could be compiled with GCC. Torvalds answered that, while he used to find problems in the LLVM Clang compiler, now he's more likely to find problems with GCC instead; he now builds with Clang. Ojeda said that he is working on finding developer resources for gccrs; the project is currently sitting on over 800 out-of-tree patches and still has a lot of work to do on top of that. GCC support will be a while, he said. 

Ts'o complained that the language still isn't entirely stable. This could be a particular problem for the confidential-computing community; they are concerned about security and, as a consequence, about backports to long-term-support kernels. But if those kernels are on different Rust versions, those backports will be problematic. Ojeda said that, while it is a "crazy idea", backporting the version upgrades could be considered. He doesn't think that the change rate will be high enough to be a problem, though. 

At the conclusion, Torvalds pointed out that there have been problems over the years with GCC changes breaking the kernel; the same will surely happen with Rust, but it will be the same thing in the end. The session, well over time, was brought to a halt at this point. Whether the kernel community has truly concluded that it is committed to Rust remains to be seen; there will almost certainly be pull requests adding significant Rust code in the near future.
