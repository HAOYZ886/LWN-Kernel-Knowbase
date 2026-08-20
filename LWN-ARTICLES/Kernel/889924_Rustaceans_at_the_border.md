---
title: Rustaceans at the border
url: https://lwn.net/Articles/889924/
date: "April 14, 2022"
category: "Development tools-Rust"
author: "By Jonathan Corbet April 14, 2022"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 14, 2022

Support for developing in the [Rust](<https://www.rust-lang.org/>) language is [headed toward the kernel](<https://lwn.net/Articles/870555/>), though just when it will land in the mainline is yet to be determined. The Rust patches are progressing, though, and beginning to attract attention from beyond the kernel community. When two languages — and two different development communities — come together, the result can be a sort of cultural clash. Some early signs of that are appearing with regard to Rust in the kernel; if the resulting impedance mismatches can be worked out, the result could be a better development environment for everybody involved. 

The latest round of Rust patches was [posted by Miguel Ojeda](<https://lwn.net/ml/linux-kernel/20220317181032.15436-1-ojeda@kernel.org/>) on March 17\. This time around, Rust support has moved forward to [version 1.59.0](<https://blog.rust-lang.org/2022/02/24/Rust-1.59.0.html>) of the Rust language, which has stabilized a couple of important (for the kernel) features. The patches add a new module abstracting access to hardware random-number generators. A `CString` type has been added for C strings. The spinlock implementation has been improved. All told, the patch series, which can be found in the linux-next repository, adds over 35,000 lines of code and documentation; it is a considerable body of work. 

There has been no public discussion on just when these patches might be deemed ready to go into the mainline kernel. Rust support is still considered "experimental" even by its developers; that is likely to remain the case for some time (even after this work is merged into the mainline) until the language proves itself for kernel development. 

Clearly, though, some developers are beginning to play with it — and they are not all traditional kernel developers. Recently, Nándor Krácser [asked](<https://lwn.net/ml/rust-for-linux/CAHkG_ewRo5uPOue3ZMAAPAc+eP7MNNU5iVym-JVG1jN7HD+XMg@mail.gmail.com/>) on the Rust-for-Linux mailing list about the possibility of including Rust modules from the [crates.io repository](<https://crates.io/>) into the kernel build. This request, seemingly, [is not just for small stuff](<https://lwn.net/ml/rust-for-linux/CAHkG_ewQ3uJBTTPQ24+faW6v2ZHvGhTR2qTpfrj6URTr6hy0SQ@mail.gmail.com/>): 

> Currently I'm experimenting with different crates which I would like to use in my module, serialization libraries, math libraries. etc, even complex ones, are really hard to pull in as a direct source library (copy the code to the module), and if they have a transitive dependency that complicates things even more. 

Shortly thereafter, Chris Suter [showed up](<https://lwn.net/ml/rust-for-linux/CAKfU0DLS5icaFn0Mve6+y9Tn1vL+eLKqfquvXbX4oCpYH+VapQ@mail.gmail.com/>) with a similar request. Rust developers working with kernel modules, it seems, want more functionality than the current `kernel` crate provides to them. 

This should not be entirely surprising. Like many newer languages, Rust is closely tied to a language-specific package-management system and associated central repository; in this case, the [Cargo](<https://doc.rust-lang.org/cargo/>) package manager and crates.io. Developers in such languages quickly become accustomed to pulling in new modules (and any dependencies they may have) with a simple command, and to having the build system make dependencies magically appear when building a new program. For these developers, the idea of working in an environment where complex libraries are not obtainable with a few keystrokes starts to have a distinct lack of appeal after a while. 

The kernel does not work in this way, though. To those of us who didn't grow up with that kind of development environment, it looks like a recipe for bloat, bugs, and security problems. Depending on central repositories opens up a project to problems like the famous [leftpad incident](<https://lwn.net/Articles/681410/>) or, worse, the deliberate insertion of malicious software. A lack of attention to API compatibility leads to a thicket of version requirements and dependency-resolution problems so complex that [machine-learning systems](<https://thoth-station.ninja/>) are emerging to deal with them. Plus it all just looks so undisciplined and messy. 

At least some of the criticisms of this mode of development are valid, but it's also not hard to detect a bit of Stockholm syndrome as well. For many of us, for much of our careers, building a new program from source was likely to involve a lengthy cycle of "try to build, figure out which dependency it wants now, install the dependency" iterations — and recursive iterations at that when the dependencies turn out to have missing dependencies of their own. This exercise helped us to understand our systems better and must somehow have helped us to build better moral character, so we can't understand why Kids These Days just don't want to live that way. 

The kernel community seems more than usually likely to have developers who are resistant to newer methods of development. The kernel has to stand alone, and its developers keep a firm grip on its dependencies. The kernel repository contains all of the code needed to build a working kernel; developers can be expected to install a limited set of tools to do the build, but the idea of installing external libraries to build into the kernel would not go over well. 

So when developers see a shopping list like the one [posted](<https://lwn.net/ml/rust-for-linux/CAKfU0DJ1AqR4cy4=706qRGESozHii9dPL5BYQV047cZkyn3RzA@mail.gmail.com/>) by Suter: 

> Like I said, I'm interested in futures. Why it's useful: async Rust is arguably more common and easier to use than other forms of multi-threaded processing. Other crates that I'd like: anyhow, bincode, byteorder, log, once_cell, pin-project, rand, serde, slab, static_assertions, uuid plus some more esoteric ones. 

The first temptation will be to either run and hide or to respond in a way that may not be compliant with anybody's code of conduct. 

There are some good reasons for this. As Greg Kroah-Hartman [pointed out](<https://lwn.net/ml/rust-for-linux/YjsHLJPb2haKbOBg@kroah.com/>), code that has been written to be useful in user space almost certainly does not work within the constraints imposed on kernel code. "Async Rust" knows nothing about kernel threads or how context switching is done in the kernel, for example. Kernel code must be extremely careful in how it allocates memory, must not use floating-point arithmetic, cannot store large data structures on the stack, and cannot use unbounded recursion, among many other rules. Most user-space code, which was not written with these rules in mind, will fare poorly in this environment. For this reason, Kroah-Hartman said that any functionality desired by Rust programs must be specially written and provided in the dedicated `kernel` crate. 

The Rust-for-Linux developers understand this situation and [are not envisioning](<https://lwn.net/ml/rust-for-linux/CAFRnB2W+8XqOj0aL5DP2cMNLO2uR0DpPk-qB-oC7r=zT5N5gGA@mail.gmail.com/>) adding the ability to pull in modules with a tool like Cargo. So it is interesting that a long-time kernel developer, Kent Overstreet, was the one to [argue](<https://lwn.net/ml/rust-for-linux/20220330204353.57w3fxtaw4wwqvi3@moria.home.lan/>) for a different approach. ""The world is changing"", he said, and perhaps it is time for the kernel community to change with it as well. There are numerous situations where it can be beneficial to run code in both user and kernel space, he said, and the fact that doing so is currently painful is a problem for developers on both sides: 

> The solution to problems like these are to stop thinking that kernelspace and userspace _have_ to be completely different beasts \- they really don't have to be! and start encouraging different thinking and tooling improvements that will make our lives easier. 

It is true that the boundary between kernel and user space has become more porous over the years. Various subsystems provide hooks that allow formerly kernel-specific tasks to be carried out in user space instead, while user space can use BPF to run code inside the kernel. But the two environments are still quite different, and code meant to run on one side generally cannot run on the other. There has not been a lot of effort put into thinking about how to reduce that divide; perhaps it really is time for that to change. The Rust language might just be the environment in which this transformation could happen. As Overstreet put it: 

> Rust's conditional compilation & generics story is _much_ better than what we've had in the past in C and C++, meaning writing Rust code that works in both userspace and kernelspace is much saner than in the past. 

If an initiative like this were to work, it could greatly reduce the barrier to entry for future kernel developers while making a lot of useful code available to the kernel community. It would be a different kernel project than the one we know now, but it might be a more fun and more productive one. 

Interesting things tend to happen when immigrants show up in a new land. They can often create a backlash among those who are already there — the new people dress funny and their cooking smells weird, after all, and some of them even have [a crab](<https://rustacean.net/>) as their mascot. But they can also bring energy and ideas that shake up their new home and make it richer for everybody involved. It may just be that we will see something like that happen if and when a crowd of Rust developers descends upon the kernel community. The end result could be difficult to recognize — and perhaps better than anything we had before.
