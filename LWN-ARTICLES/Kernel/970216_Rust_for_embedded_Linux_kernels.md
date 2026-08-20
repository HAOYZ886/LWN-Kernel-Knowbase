---
title: Rust for embedded Linux kernels
url: https://lwn.net/Articles/970216/
date: "April 23, 2024"
category: "Development tools-Rust"
author: "By Jonathan Corbet April 23, 2024 OSSNA"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 23, 2024

* * *

[OSSNA](<https://lwn.net/Archives/ConferenceByYear/#2024-Open_Source_Summit_North_America>)

The Rust programming language, it is hoped, will bring a new level of safety to the Linux kernel. At the moment, though, there are still a number of impediments to getting useful Rust code into the kernel. In the Embedded Open Source Summit track of the [Open Source Summit North America](<https://events.linuxfoundation.org/open-source-summit-north-america/>), Fabien Parent provided an overview of his work aimed at improving the infrastructure needed to write the device drivers needed by embedded systems in Rust; there is still some work to be done. 

Parent started with the case for using Rust in the kernel; it may not be a proper justification, he said, but it is true that Rust is one of the most admired languages in use. C is about 50 years old and has not changed much since the C89 standard came out. It has the advantage of a simple syntax that is easy to learn, and it is efficient for writing low-level code. But C also makes it easy to write code containing undefined behavior and lacks memory-management features. 

[![\[Fabien Parent\]](https://static.lwn.net/images/conf/2024/ossna/FabienParent-sm.png)](<https://lwn.net/Articles/970222/>) Rust, instead, is about ten years old and has a new release every six weeks. It is harder to learn and forces developers to come up to speed on concepts like ownership and borrowing. But the code produced is efficient; Rust's abstractions are meant to be zero-cost, with the verification work done at compile time. Rust forces developers to handle errors, eliminating another frequent cause of bugs. 

Thus, he said, it makes sense to use Rust in the kernel, hopefully leading to safer code overall. There is basic Rust support in the kernel now, but it is focused on driver code. There is currently no plan to support core-kernel code written in Rust, partly because the LLVM-based rustc compiler, which is the only viable compiler for Rust code currently, does not support all of the architectures that the kernel does. Rust support in the kernel is still considered to be experimental. 

There are some drawbacks to using Rust in the kernel, starting with the current drivers-only policy. Most kernel vulnerabilities, he said, are not actually in driver code; instead, they appear in core code like networking and filesystems. As long as Rust is not usable there, it cannot help address these problems. Adding Rust, of course, will complicate the maintenance of the kernel, forcing maintainers to learn another language. The abstractions needed to interface Rust to the rest of the kernel are all new code, some of which may well contain bugs of its own. 

Parent became interested in Rust after stumbling across [a sample GPIO driver](<https://lwn.net/Articles/863459/>) in Rust on LWN. He immediately started trying to write some kernel code in Rust, but failed soon thereafter. At this point, there simply is not a lot of kernel code that a new developer can use to learn from. So, instead, he went and rewrote all of his custom tools in Rust; after that, he was better prepared to work on the kernel. 

There are, he said, a lot of people trying to contribute to the [Rust-for-Linux](<https://rust-for-linux.com/>) effort; there is [an online registry](<https://github.com/tgross35/RFL-patch-registry>) containing much of that work. But many of the basic abstractions needed for useful Rust code still are not in the mainline, and that is preventing others from making progress. The work that _is_ seemingly advancing, including support for graphics drivers, [Android's binder](<https://lwn.net/Articles/953116/>), and filesystems like [PuzzleFS](<https://lwn.net/Articles/945320/>) are not useful for the embedded work that Parent is interested in. Most of this work has been done on x86 systems, with the exception of the Apple M1 GPU driver. Many of the key abstractions needed for embedded work are missing from the kernel; many of those exist, but they are often unmaintained. 

Parent had a long list of requirements for embedded systems, starting with support for the Rust language on 64-bit Arm systems; that, at least, has been merged for the upcoming 6.9 kernel release. Many abstractions for subsystems like clocks, pin control, run-time power management, regulators, and so on are not yet there. The abstractions have proved to be a challenge; maintainers will not merge code that is not used elsewhere in the kernel, but drivers cannot be merged until the abstractions are there. That leads to a situation where a lot of people are involved, each of whom are waiting on pieces from the others. That makes it hard to get the pieces upstream. 

Parent's objective is to write simple drivers with minimal dependencies, each of which can be used to get a small number of abstractions upstream. He gave as an example a [regulator driver](<https://github.com/Fabo/linux/commit/140c7b2e7f4716edcbb80d5cf5ee04f18cab4def>) that needs a relatively small set of abstractions, including those for platform drivers, regulators, regmap, I2C drivers, and Open Firmware for probing. He will be trying to get that set upstream; from there, work can proceed to more complex drivers. 

The (conspicuously undocumented) regmap interface was called out for how it can showcase the advantages of Rust. Regmap eases access to devices that export an array of registers for configuration and operation. The Rust regmap abstraction allows the provision of a type-safe interface, built on top of the [regmap_field API](<https://patchwork.kernel.org/project/linux-arm-kernel/patch/1370856118-6503-1-git-send-email-srinivas.kandagatla@st.com/>), that is generated with some "macro magic". The type checking allows the interface to ensure that register operations use the correct data types with each register, catching a number of common errors. 

Parent's next step is to upstream a lot of this work, a task that, he acknowledges, will be difficult. But, if nothing else, he has learned a few lessons, starting with the fact that abstractions are more complex than one might expect, and they will have bugs. One problematic area is in ownership of resources; that is going to be hard to nail down for as long as there are extensive interfaces between the Rust and C sides. He advised other Rust developers to not try to write complete abstractions at the outset; instead, only the parts that are actually needed should be implemented. 

Linked lists, a famous point of difficulty for Rust in general, present a special hazard in kernel code. The Rust compiler likes to move data around as a program runs; if that data happens to be a structure containing linked-list pointers, moving it will break the list and create hard-to-find bugs. Adding a `list_head` structure to an existing C structure can, as a result, break a Rust abstraction built on that structure in ways that are hard to detect automatically. The way he talked about this problem suggested a certain amount of hard-earned experience. 

Even so, he summarized, writing kernel code in Rust makes a lot of things easier. Error handling is much more straightforward, and the compiler can ensure that developers have handled all possible values. Driver code tends to be a lot shorter and, he said, if the code compiles, it is likely to work. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting our travel to this event.]
