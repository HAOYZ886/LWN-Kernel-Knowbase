---
title: Rust for Linux redux
url: https://lwn.net/Articles/862018/
date: "July 7, 2021"
category: "Development tools-Rust"
author: "By Jake Edge July 7, 2021"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
July 7, 2021

On July 4, the [Rust for Linux](<https://github.com/Rust-for-Linux>) project [posted](<https://lwn.net/ml/linux-kernel/20210704202756.29107-1-ojeda@kernel.org/>) another version of its patch set adding support for the language to the kernel. It would seem that the project feels that it is ready to be considered for merging into the mainline. Perhaps a bigger question lingers, though: is the kernel development community ready for Rust? That part still seems to be up in the air. 

#### Round 2

Miguel Ojeda, who has been hired to [work on the project full-time](<https://lwn.net/Articles/860160/>), posted the patch set; in the cover letter, he listed a number of changes and updates since the [RFC patch set](<https://lwn.net/ml/linux-kernel/20210414184604.23473-1-ojeda@kernel.org/>) was [posted back in April](<https://lwn.net/Articles/853423/>). In particular, the allocations that would call `panic!()` when they failed have been replaced. A modified version of the [`alloc` crate](<https://doc.rust-lang.org/alloc/>) has been [created for the kernel project](<https://rust-for-linux.github.io/docs/alloc/>), though the plan is for the changes to work their way into the upstream project so that the customized version can eventually be dropped. Incidentally, the patch adding the modified crate was [apparently too large for the lore archive](<https://lwn.net/ml/linux-kernel/CANiq72nQq8Y8v9Pyf7JFq6Kf-+doNP+mHAFNzj_cSFBa3KwS5w@mail.gmail.com/>), though it does [show up](<https://lwn.net/ml/linux-kernel/20210704202756.29107-8-ojeda@kernel.org/>) in the LWN archive. 

There is more progress on Rust abstractions for kernel facilities, including ""red-black trees, reference-counted objects, file descriptor creation, tasks, files, io vectors..."", he said, as well as additions for driver support. Beyond that, the Rust driver for the [Android Binder](<https://elinux.org/Android_Binder>) interprocess communication mechanism has more features and there is ongoing work on a driver for the hardware random-number generator available on some Raspberry Pi models. 

But the lack of a "real" driver was one of the sticking points back in April, and it was raised again this time. Christoph Hellwig [said](<https://lwn.net/ml/linux-kernel/YOVNJuA0ojmeLvKa@infradead.org/>) that he was strongly against merging the code if that was the intent of the posting. In the cover letter, Ojeda said that the patch set was being added to linux-next, and he [confirmed](<https://lwn.net/ml/linux-kernel/CANiq72mKPFtB4CtHcc94a_y1V4bEOXXN2CwttQFvyzwXJv62kw@mail.gmail.com/>) that it is being submitted for merging. Hellwig wants to see Rust prove itself before being included: 

> [...] prove it actually is useful first. Where useful would be a real-life driver like say nvme or a usb host controller driver that actually works and shows benefits over the existing one. Until it shows such usefulness it will just drag everyone else down. 

Ojeda pointed at the Binder driver as a ""non-trivial module"" that ""is already working"", but acknowledged that a real hardware driver is an important (and, as yet, missing) piece. But the cover letter listed a number of organizations that are interested in or involved with the project already, and Ojeda believes that is also something that is helping to prove the project: 

> It is "proven" in the sense we are already starting to get users downstream and other interested parties have shown support. 
> 
> However, others are more conservative and will only start investing into it if we are in mainline, i.e. if the decision of having Rust or not is already taken. 

But the Binder "driver" is not really a good example to use for a few different reasons, Greg Kroah-Hartman [said](<https://lwn.net/ml/linux-kernel/YOXB7FRqldZik2Xn@kroah.com/>). It is missing a fairly large piece of functionality (binderfs) for one thing, but it also does little to help show how Rust will fit in with the rest of the kernel. As before, he strongly recommended working on something that would help clear up some of the questions that kernel developers have about Rust: 

> Not to say that it doesn't have its usages, but the interactions between binder and the rest of the kernel are very small and specific. Something that almost no one else will ever write again. 
> 
> Please work on a real driver to help prove, or disprove, that this all is going to be able to work properly. There are huge unanswered questions that need to be resolved that you will run into when you do such a thing. 

#### Kernel Summit topic

The theme of Rust proving itself was also present in a thread on the ksummit-discuss mailing list. At the end of June, Ojeda [proposed](<https://lwn.net/ml/ksummit-discuss/CANiq72kF7AbiJCTHca4A0CxDDJU90j89uh80S3pDqDt7-jthOg@mail.gmail.com/>) Rust for Linux as a technical topic for the Kernel Summit track at this year's [Linux Plumbers Conference](<https://linuxplumbersconf.org/>). On July 6, Linus Walleij [replied](<https://lwn.net/ml/ksummit-discuss/CACRpkdbbPEnNTLYSP-YP+hTnqhUGQ8FjJLNY_fpSNWWd8tCFTQ@mail.gmail.com/>), agreeing that it was a topic that should be discussed. He noted that there are already quite a few languages that kernel developers need to be up on (e.g. C, assembly, Make, Bash, Perl, Python, ...), so a question in his mind is what Rust will bring to the table that makes it worth adding to the list. 

That message started the ball rolling on a discussion of how Rust fits—and where the experiment leads. There is clearly some concern among kernel developers about having to know Rust to continue working on the kernel. Ojeda tried to allay those fears to a certain extent, but did acknowledge that most kernel developers would eventually need to understand the language. For now, the intent is for the project to work with maintainers that want to add a Rust API to their subsystem, Ojeda [said](<https://lwn.net/ml/ksummit-discuss/CANiq72mPMa9CwprrkL7QsEChQPMNtC61kJgaM4Rx0EyuQmvs2g@mail.gmail.com/>). That will bootstrap support for Rust within the kernel and will help alleviate some of the concerns expressed by [ Leon Romanovsky](<https://lwn.net/ml/ksummit-discuss/YOPcZE+WjlwNueTa@unreal/>), [James Bottomley](<https://lwn.net/ml/ksummit-discuss/19e0f737a3e58ed32758fb4758393c197437e8de.camel@HansenPartnership.com/>), and others. 

Romanovsky was worried about refactoring, especially cross-subsystem refactoring, in a system with drivers in both languages. Bottomley suggested that there would be fewer reviewers for the Rust code, so bugs could more easily slip in: 

> Since most of our CVE type problems are usually programming mistakes nowadays, the lack of review could contribute to an increase in programming fault type bugs which aren't forbidden by the safer memory model. 

In the near term, those are problems that will need to be dealt with, but the goal is to get to a point where Rust knowledge is more widespread among kernel developers. As Ojeda [put it](<https://lwn.net/ml/ksummit-discuss/CANiq72=vjXYN-A1gZysXzKvR+NgmxpSGOiOGro0S6tMhYAwR6Q@mail.gmail.com/>): 

> After all, if we are going to have Rust as a second language in the kernel, we should try to have as many people on board as possible, at least to some degree, within some reasonable time frame. 

As Laurent Pinchart [pointed out](<https://lwn.net/ml/ksummit-discuss/YOSjETKWhuRz0Poq@pendragon.ideasonboard.com/>), that is an important point and one that needs to be more clearly highlighted in the discussion: 

> [...] adopting Rust as a second language in the kernel isn't just a technical decision with limited impact, but also a process decision that will create a requirement for most kernel developers to learn Rust. Whether that should and will happen is what we're debating, but regardless of the outcome, it's important to phrase the question correctly, with a broad view of the implications. 

There are, of course, plenty of technical hurdles that need to be cleared as well. One of those areas is the Linux driver model and the object-lifetime handling that goes along with it. Roland Dreier [suggested](<https://lwn.net/ml/ksummit-discuss/CAG4TOxMzf1Wn6PcWk=XfB+SV+MHwbxUq8t1RNswie5e3=Y+OXQ@mail.gmail.com/>) that interfaces like [devres](<https://www.kernel.org/doc/html/latest/driver-api/driver-model/devres.html>) (i.e. managed device resources) could have avoided a lot of problems that he has seen in drivers regarding lifetime management, and error paths in particular, but that it has not been adopted widely. Walleij [disagreed](<https://lwn.net/ml/ksummit-discuss/CACRpkdZyJd0TW5aVRfxSSWknzCyVhjMwQuAj9i9iuQ6pW9vftQ@mail.gmail.com/>) that it hasn't been widely used, but is not at all sure that a Rust switch is better: 

> I think it's a formidable success, people just need to learn to do it more. 
> 
> But if an easier path to learn better behaviours is to shuffle the whole chessboard and replace it with drivers written in Rust, I don't know? Maybe? 

For Kroah-Hartman, seeing real drivers in Rust for the kernel would [help clear up](<https://lwn.net/ml/ksummit-discuss/YOVbsS9evoCx0isz@kroah.com/>) how these problems are going to be solved in the language. There are some difficult problems that will need to be tackled: 

> This is going to be the "interesting" part of the rust work, where it has to figure out how to map the reference counted objects that we currently have in the driver model across to rust-controlled objects and keep everything in sync properly. 
> 
> For the normal code, the fact that the memory was assigned to one specific object (the struct device) but yet referenced from another object (the cdev). devm_* users like this do not seem to realize there are two separate object lifecycles happening here as the interactions are subtle at times. 
> 
> I am looking forward to how the rust implementation is going to handle all of this as I have no idea. 

There is also the question of where all of this leads. Walleij [wondered](<https://lwn.net/ml/ksummit-discuss/CACRpkdatK-U16oefmGoov7u_obekVP4m1fT=6joCVpT00Sm09A@mail.gmail.com/>) whether the "leaf nodes" of the kernel (i.e. drivers) are actually the best place to start from the perspective of showing the benefits of the language. He also [asked](<https://lwn.net/ml/linux-kernel/CACRpkdat-4BbKHMBerdxXBseMb9O3PiDRZmMLP_OWFE2ctSgEg@mail.gmail.com/>) about that in a lengthy message back in April. Part of his question is whether the decision to start with drivers was made for other reasons: 

> If the whole rationale with doing device drivers first is strategic, not necessarily bringing any benefits to that device driver subsystem but rather serving as a testing ground and guinea pig, then that strategy needs to be explicit and understood by everyone. So while we understand that Rust as all these $UPSIDES doing device drivers first is purely strategic, correct? I think the ability to back out the whole thing if it wasn't working out was mentioned too? 

He also asked about writing whole subsystems in Rust. That would entail exposing Rust APIs to C code elsewhere in the kernel. It would also potentially allow an evolution toward more and more Rust in the kernel: 

> If we want to *write* a subsystem in Rust then of course it will go the other way: Rust need to expose APIs to C. And I assume the grand vision is that after that Rust will eat Linux, one piece at a time. If it proves better than C, that is. 

Ojeda [said](<https://lwn.net/ml/ksummit-discuss/CANiq72mi-oGAgsws5SpbVyp+NOPXp_6NYq-oWLKe_UD95POGVA@mail.gmail.com/>) that while it is possible to expose Rust APIs to the C part of the system, he does not recommend it. The problem is that the C callers lose much of the benefit that Rust brings to the table: 

> In general, I would avoid exposing Rust subsystems to C. 
> 
> It is possible, of course, and it gives you the advantages of Rust in the *implementation* of the subsystem. However, by having to expose a C API, you would be losing most of the advantages of the richer type system, plus the guarantees that Rust bring as a language for the consumers of the subsystem. 

In a similar vein, Johannes Berg [asked](<https://lwn.net/ml/ksummit-discuss/db6ee5da45bbf526b13fda7d1cd2bf93f24cd84f.camel@sipsolutions.net/>) about replacing parts of a subsystem with Rust, but leaving the drivers in C—the reverse of the existing plan, effectively. Once again, Ojeda [said](<https://lwn.net/ml/ksummit-discuss/CANiq72=HAMqn6aqVpieyLyP4gOMT4zKzYRNTZ0TqNYEC4hdjKw@mail.gmail.com/>) that its possible, but cautioned about losing Rust features and guarantees. In addition, there are architectures where there is no Rust compiler available at this point, so it may be premature to be looking at Rust-based subsystems. 

#### Future

The project's eventual goal is not entirely clear. If all of the Linux drivers were written in Rust, there would still be lots of big important pieces that are running unsafe C; is the next step to replace those if Rust proves itself in the meantime? But it is also unclear what "proves itself" actually means in this context. 

Learning a new language, with all of its different behaviors, quirks, and idiosyncrasies, is a pretty big ask for developers who are already juggling a fair amount of complexity in maintaining the existing kernel code. Not to mention the added complexity of providing new functionality on top of what's already there. The skepticism that seems fairly prevalent from commenters in both threads (and elsewhere) likely stems from that learning burden. 

Adding Rust to the kernel requires a lot of work, for a lot of different people, without a clear and obvious benefit beyond promises that can only truly be fully fulfilled with a whole kernel written in Rust. With luck, the project can provide some clear "wins" in the early going that clearly demonstrate both the potential of the language and an ability to be utilized incrementally on a large and complex code base like the kernel. Without that, it may be difficult for the project to progress very far in the goal of "rustifying" the kernel.
