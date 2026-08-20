---
title: "Rust heads into the kernel?"
url: https://lwn.net/Articles/853423/
date: "April 21, 2021"
category: "Development tools-Rust"
author: "By Jake Edge April 21, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
April 21, 2021

In a [lengthy message](<https://lwn.net/ml/linux-kernel/20210414184604.23473-1-ojeda@kernel.org/>) to the linux-kernel mailing list, Miguel Ojeda "introduced" the [Rust for Linux](<https://github.com/Rust-for-Linux>) project. It was likely not the first time that most kernel developers had heard of the effort; there was an [extensive discussion](<https://lwn.net/Articles/829858/>) of the project at the [2020 Linux Plumbers Conference](<https://www.linuxplumbersconf.org/event/7/>), for example. It has also been [raised before](<https://lwn.net/ml/linux-kernel/CAKwvOdmuYc8rW_H4aQG4DsJzho=F+djd68fp7mzmBp3-wY--Uw@mail.gmail.com/>) on the list. Now, the project is looking for feedback from the kernel community about its plans, thus the RFC posting on April 14\. 

#### Adding Rust

Ojeda started by acknowledging that adding another implementation language to the kernel will be at least somewhat disruptive, so there need to be good reasons to do so. The kernel is already a highly complex body of code to understand and work with, adding a second language into the mix, with all of its complexities, only makes that worse. ""Nevertheless, we believe that, even today, the advantages of using Rust outweighs the cost."" 

Those benefits mainly stem from the [memory-safety](<https://en.wikipedia.org/wiki/Memory_safety>) features of the Rust language. The hope is that the number of bugs in the kernel can be reduced by eliminating these kinds of problems, at least for the pieces that get implemented in Rust. For now, those pieces are envisioned to be well away from the core kernel code: 

> Please note that the Rust support is intended to enable writing drivers and similar "leaf" modules in Rust, at least for the foreseeable future. In particular, we do not intend to rewrite the kernel core nor the major kernel subsystems (e.g. `kernel/`, `mm/`, `sched/`...). Instead, the Rust support is built on top of those. 

There are lots of impedance mismatches between kernel C code and Rust that need to be handled—one way or another. In order to get the most benefit from Rust's safety features, the amount of kernel support code in [`unsafe` blocks](<https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html>) needs to be minimized—and carefully documented. One of the stated goals is that the documentation guidelines be automatically enforced: 

> By taking advantage of Rust tooling, we keep enforcing the documentation guidelines we have established so far in the project. For instance, we require having all public APIs, safety preconditions, `unsafe` blocks and type invariants documented. 

The work so far has focused on making the building blocks and starting to implement wrappers for kernel APIs and abstractions, but there is lots more to do: ""Covering the entire API surface of the kernel will take a long time to develop and mature."" The RFC further acknowledges the task ahead, but notes that there is tooling available to help that process: 

> [...] modules written in Rust should never use the C kernel APIs directly. The whole point of using Rust in the kernel is that we develop safe abstractions so that modules are easier to reason about and, therefore, to review, refactor, etc. 
> 
> Furthermore, the bindings to the C side of the kernel are generated on-the-fly via `bindgen` (an official Rust tool). Using it allows us to avoid the need to update the bindings on the Rust side. 

In a [Google security blog post](<https://security.googleblog.com/2021/04/rust-in-linux-kernel.html>), which led to a lengthy comment stream when posted [here at LWN](<https://lwn.net/Articles/852704/>), one of the Rust for Linux maintainers, Wedson Almeida Filho, gave a detailed description of one of the example drivers that are part of Ojeda's RFC. It is a [character device that implements a semaphore](<https://github.com/Rust-for-Linux/linux/blob/e2382072517dd0b6b65d4e179e322f392efb9676/samples/rust/rust_semaphore.rs>), mostly for demonstration purposes. The RFC also has a reimplementation of the [Android Binder](<https://elinux.org/Android_Binder>) interprocess communication mechanism. While the latter is not yet complete, it gives a further look at what would be possible with Rust in the kernel: 

> At the moment we have nearly all generic kernel functionality needed by Binder neatly wrapped in safe Rust abstractions [...] 
> 
> We also continue to make progress on our Binder prototype, implement additional abstractions, and smooth out some rough edges. This is an exciting time and a rare opportunity to potentially influence how the Linux kernel is developed, as well as inform the evolution of the Rust language. 

The RFC notes that, currently, the Rust support adds a fair amount of code to the built kernel, but that there are plans to reduce that over time. The kernel size for ""small x86_64 config we use in the CI"" increased by around 4% with full Rust support. The Rust version of the semaphore driver is around 50% bigger than its C counterpart, while the Binder driver is roughly equivalent in size. However, ""note that while the Rust version is not equivalent to the C original module yet, it is close enough to provide a rough estimation"". 

#### Reaction

Overall, the reception to the RFC was favorable, though there are some exceptions, and, of course, there are questions and concerns with the existing code. Linus Torvalds seemed to focus in on the `BUG()` calls in the support code and was predictably unhappy to see them. The kernel tries hard to continue even in the face of errors, but calling `BUG()` simply gives up and crashes the kernel with a backtrace. [In one case](<https://lwn.net/ml/linux-kernel/CAHk-=wgJvJJtd2mpYpx5+zn_hPrSOqGqi-Pxb7e+h+anhsLnQg@mail.gmail.com/>), there are intrinsic operations (""panicking intrinsics"") included in the Rust standard library that are not supported for the kernel; calling them effectively crashes the kernel by calling `BUG()`. Torvalds suggested making calls to those fail at build time; Ojeda [agreed](<https://lwn.net/ml/linux-kernel/CANiq72=PakH-25ZAhC-TKNq7ti76cKt-coS4UahUjsGxNVWRAQ@mail.gmail.com/>) that would be a better approach, but also noted that more of the standard library will be removed over time, which may largely eliminate the problem. 

Torvalds also [pointed](<https://lwn.net/ml/linux-kernel/CAHk-=wiVY56LzwV_G075NEFwsdf-p7GOTy_cB7-UU9b=49rB1g@mail.gmail.com/>) to the `panic!()` calls in the memory allocation code, which seemed ""fundamentally wrong"" to him: 

> If the Rust compiler ends up doing hidden allocations, and they then cause panics, then one of the main *points* of Rustification is entirely broken. That's 100% the opposite of being memory-safe at build time. 
> 
> An allocation failure in some random driver must never ever be something that the compiler just turns into a panic. It must be something that is caught and handled synchronously and results in an ENOMEM error return. 

Again, Ojeda [agreed](<https://lwn.net/ml/linux-kernel/CANiq72=VnWEdh2uuwH2JM6ZhrEVDiQisnNBQGX-qwPnTitcR=g@mail.gmail.com/>); he noted that there is work to do to adapt Rust's standard library for use by the kernel: 

> What happens here is that we use, for the moment, `alloc`, which is part of the Rust standard library. However, we will be customizing/rewriting `alloc` as needed to customize its types (things like `Box`, `Vec`, etc.) so that we can do things like pass allocation flags, ensure we always have fallible allocations, perhaps reuse some of the kernel data structures, etc. 

Using the Binder driver as an example was [another area of concern for Torvalds](<https://lwn.net/ml/linux-kernel/CAHk-=wjdZ1KksHHHuekeAx9kKFXEyt+rg0P=yRD1Bia_01wucg@mail.gmail.com/>); he would like to see an ""example of a real piece of code that actually does something meaningful"". Ojeda [said](<https://lwn.net/ml/linux-kernel/CANiq72mpjQh01QovUPCjdHDS-zVHroxymet67GOHvotW8wGdVg@mail.gmail.com/>) there are plans to add a few drivers that talk to real hardware; Matthew Wilcox had an [idea for where to start](<https://lwn.net/ml/linux-kernel/20210414194946.GW2531743@casper.infradead.org/>): 

> I'd suggest NVMe as a target. It's readily available, both as real hardware and in (eg) qemu. The spec is freely available, and most devices come pretty close to conforming to the spec until you start to push hard at the edges. Also then you can do performance tests and see where you might want to focus performance efforts. 

Greg Kroah-Hartman [agreed with Torvalds](<https://lwn.net/ml/linux-kernel/YHfm+tCYp+I1C5I%2F@kroah.com/>) that Binder did not make a particularly good example driver, but was duly impressed with what the project had accomplished so far: 

> [...] this patchset is a great start that provides the core "here's how to build rust in the kernel build system", which was a non-trivial engineering effort. Hats off to them that "all" I had to do was successfully install the proper rust compiler on my system (not these developers fault), and then building the kernel code here did "just work". That's a major achievement. 

He also thought that NVMe might make a good choice, but had other thoughts ""for some of the basics that driver authors deal with on a daily basis (platform driver, gpio driver, pcspkr driver, /dev/zero replacement)"". It would seem that the Rust for Linux project will be working on one or more of these kinds of "real" drivers before long. 

While he reiterated the complaints he had for some of the individual patches, Torvalds [said](<https://lwn.net/ml/linux-kernel/CAHk-=wh_sNLoz84AUUzuqXEsYH35u=8HV3vK-jbRbJ_B-JjGrg@mail.gmail.com/>): ""on the whole I don't hate it"". On the other hand, though, Peter Zijlstra seemed to [fundamentally object](<https://lwn.net/ml/linux-kernel/YHiMyE4E1ViDcVPi@hirez.programming.kicks-ass.net/>) to the idea of adding a second implementation language to the kernel. The RFC noted that the kernel tooling has been focused on C, ""including compiler plugins, sanitizers, Coccinelle, lockdep, sparse"", but that tooling for Rust will ""likely improve if Rust usage in the kernel grows over time"". Zijlstra zeroed in on that and asked: 

> This; can we mercilessly break the .rs bits when refactoring? What happens the moment we cannot boot x86_64 without Rust crap on? 
> 
> We can ignore this as a future problem, but I think it's only fair to discuss now. I really don't care for that future, and IMO adding this Rust or any other second language is a fail. 

Perhaps unsurprisingly, he has strong opinions against the documentation format used by the project (Markdown in the code that gets converted [to HTML](<https://rust-for-linux.github.io/docs/kernel/>)). He was also unhappy with the code formatting used, which follows, at least for now, ""Rust's idiomatic style"", according the RFC, but is ""really *really* hard to read"". Beyond those, he wondered about what memory model Rust follows and how it ""aligns (or not)"" with the [Linux kernel memory model](<https://lwn.net/Articles/718628/>) (LKMM). 

#### Memory model

Rust currently uses the C11 memory model, Boqun Feng [said](<https://lwn.net/ml/linux-kernel/YHkSO3TUktyPs4Nz@boqun-archlinux/>), mostly because the LLVM compiler supports it by default, but there is interest in ensuring that its memory model works well with the kernel's. Right now, ""there is no code requiring synchronization between C side and Rust side, so we are currently fine"", but that will change eventually, so there are plans to put the right Rust and kernel people together to discuss the issue. Almeida [noted](<https://lwn.net/ml/linux-kernel/YHj02M3jMSweoP4l@google.com/>) that the plan is for most Rust code in the kernel to only need to be concerned with the Rust memory model: 

> We don't intend to directly expose C data structures to Rust code (outside the kernel crate). Instead, we intend to provide wrappers that expose safe interfaces even though the implementation may use unsafe blocks. So we expect the vast majority of Rust code to just care about the Rust memory model. 
> 
> We admittedly don't have a huge number of wrappers yet, but we do have enough to implement most of Binder and so far it's been ok. We do intend to eventually cover other classes of drivers that may unveil unforeseen difficulties, we'll see. 

Almeida disagreed with Zijlstra's characterization of HTML as being an invalid documentation format, writing that off as a personal preference. For the code formatting, he is not opposed to moving away from the Rust style if there are good reasons to do so, but found Zijlstra's criticism unconvincing: ""'Not having parentheses around the if-clause expression is complete rubbish' doesn't sound like a good reason to me."" 

Al Viro [tried to explain](<https://lwn.net/ml/linux-kernel/YHkRvhIeO2794f7v@zeniv-ca.linux.org.uk/>) the aversion to HTML documentation, in characteristically blunt fashion, which sent things briefly off the rails. Zijlstra [said](<https://lwn.net/ml/linux-kernel/YHk4DZE1ZWTiBB1f@hirez.programming.kicks-ass.net/>) that there is no real way to look at HTML documentation in ASCII; ""Nothing beats a sane ASCII document with possibly, where really needed some ASCII art."" He also explained why his seemingly arbitrary complaints about the formatting actually matter: 

> Of course it does; my internal lexer keeps screaming syntax error at me; how am I going to understand code when I can't sanely read it? 
> 
> The more you make it look like (Kernel) C, the easier it is for us C people to actually read. My eyes have been reading C for almost 30 years by now, they have a lexer built in the optical nerve; reading something that looks vaguely like C but is definitely not C is an utterly painful experience. 
> 
> You're asking to join us, not the other way around. I'm fine in a world without Rust. 

Zijlstra also [suggested](<https://lwn.net/ml/linux-kernel/YHlz54rd1YQHsOA%2F@hirez.programming.kicks-ass.net/>) that many of the Rust features being touted could be implemented in C. Almeida [agreed](<https://lwn.net/ml/linux-kernel/YHmMJWmzz2vZ3qQH@google.com/>) that they could be, but that Rust makes it impossible to mistakenly fail to use them, unlike C (at least without compiler changes): 

> In Rust, this isn't possible: the data protected by a lock is only accessible when the lock is locked. So developers cannot accidentally make mistakes of this kind. And since the enforcement happens at compile time, there is no runtime cost. 

He also raised the problem of ownership in C: there is no way to transfer an object's ownership in C, but it is straightforward to do in Rust: 

> In Rust, there is a clean idiomatic way of transferring ownership of a guard (or any other object) such that the previous owner cannot continue to use it after ownership is transferred. Again, this is enforced at compile time. 

But Zijlstra would [rather see a C extension](<https://lwn.net/ml/linux-kernel/YHmc2+bKQJ%2FXAATF@hirez.programming.kicks-ass.net/>) that supported ownership, instead of adding Rust to the kernel. 

> This would mean a far more aggressive push for newer C compilers than we've ever done before, but at least it would all still be a single language. Conversion to the new stuff can be done gradually and where it makes sense and new extensions can be evaluated on performance impact etc. 

Almeida [was not opposed](<https://lwn.net/ml/linux-kernel/YHmuX1NA5RF7C7XS@google.com/>) to that idea, quite the reverse, in fact: 

> I encourage you to pursue this. We'd all benefit from better C. I'd be happy to review and provide feedback on proposed extensions that are deemed equivalent/better than what Rust offers. 
> 
> My background is also in C. I'm no Rust fanboy, I'm just taking what I think is a pragmatic view of the available options. 

It is a little hard to imagine the kernel switching to a C extension that does not yet exist in order to avoid further investigating adding Rust into the mix, however. But there is still plenty of work that needs to be done by Rust for Linux, some of which seems likely to be needed before Torvalds would be willing to merge the support. For example, more "real" driver examples and removing the paths that lead to `BUG()` calls seem needed. But Rust for Linux is clearly getting closer to being a reality.
