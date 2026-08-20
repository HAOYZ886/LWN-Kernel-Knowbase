---
title: Committing to Rust in the kernel
url: https://lwn.net/Articles/991062/
date: "September 24, 2024"
category: "Development tools-Rust"
author: "By Jonathan Corbet September 24, 2024 Maintainers Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 24, 2024

* * *

[Maintainers Summit](<https://lwn.net/Articles/990740/>)

The project to enable the writing of kernel code in Rust has been underway for several years, and each kernel release includes more Rust code. Even so, some developers have expressed frustration at the time it takes to get new functionality merged, and an air of uncertainty still hangs over the project. At the 2024 Maintainers Summit, Miguel Ojeda led a discussion on the status of Rust in the kernel and whether the time had come to stop considering it an experimental project. There were not answers to all of the questions, but it seems clear that Rust in the kernel will continue steaming ahead. 

[![\[Miguel Ojeda\]](https://static.lwn.net/images/conf/2024/ms/MiguelOjeda-sm.png)](<https://lwn.net/Articles/991197/>) Ojeda started with the topic of the flexibility needed from the kernel's subsystem maintainers. Two years ago, before the initial Rust support was pulled into the kernel, he had requested that flexibility because there would be the need to change some core APIs at times to fit Rust code in. The need for that flexibility is being felt now, he said. 

There are some clear differences in the expectations around Rust in the kernel, he continued. He has read through thousands of comments and emails on recent events, and has seen a wide range of opinions on the state of the Rust-for-Linux project and where it is headed. It would be a good thing to converge on a common understanding of what the goals are. People and companies want to invest in Rust for the kernel, but they are unsure about its future. 

Jason Gunthorpe said that he, like many other kernel developers, has not participated in this work so far. The project was intended to demonstrate that Rust is suitable for kernel usage; he is waiting for the decision on the outcome. Dave Airlie said that the experiment is not complete, but Greg Kroah-Hartman said that it is clear at this point that Rust in the kernel is viable. Part of the reason for the apparent slow pace of the work, he said, is that the Rust developers are concentrating on device drivers; since drivers must interface with many other kernel subsystems, there is a lot of support code that must be merged. That takes time. 

Gunthorpe said that he would like to see a clear message that Rust is a success before jumping into it; he also is unable to work with Rust until a suitable compiler is available in RHEL. Airlie said that perhaps, for Gunthorpe, the time had not yet come. 

#### Tooling and help

Arnd Bergmann said that there was no doubt that drivers written in Rust would be better than those in C, but he wondered how long it would take to merge support in all the necessary subsystems, and when the tooling would be widely available. When, he asked, will he be able to build kernel code with a Rust compiler shipped by his distribution? Ojeda answered that multiple compiler versions are now supported by the kernel code, and that suitable compilers are available from many community-oriented distributions. Airlie said that it is too soon to ask the Rust community for a completely stable compiler to build kernel code with; there just is not yet enough Rust code in the kernel to make that happen. 

Linus Torvalds admonished the group that he did not want to talk about every subsystem supporting Rust at this time; getting support into some of them is sufficient for now. When Airlie asked what would happen when some subsystem blocks progress, Torvalds answered "that's my job". Christian Brauner said that the [binder driver](<https://lwn.net/Articles/953116/>) is motivating much of the subsystem work now, including the somewhat contentious [filesystem abstractions](<https://lwn.net/Articles/978738/>). That code is being reviewed now. Airlie added that the first real driver to be merged will be a sort of inflection point, after which the pace will pick up; the next challenge after that will be the creation of Rust infrastructure that is callable from C. 

Will Deacon asked Ojeda about the support that the Rust community was offering to kernel developers; Ojeda answered that he has been building a team of experts to help where needed. Some of these people are core Rust developers who know the language thoroughly; they can help to review patches even if they lack deep kernel experience. 

Torvalds pointed out that there are kernel features that are currently incompatible with Rust; that is impeding Rust support overall. He mentioned modversions in particular; that problem [is being worked on](<https://lwn.net/Articles/986892/>). The list of blocking features is getting shorter, he said, but it still includes kernel features that people need. 

#### Managing expectations

Dan Williams pointed out that he once spent two years just getting a new `mmap()` flag merged. It is necessary to manage expectations on the Rust side, he said; merging all of that code will be a slow process. Ojeda acknowledged this point, but said that the companies funding the Rust work are not seeing it going upstream; that is making them reluctant to continue that funding going forward. 

Brauner said that nobody has ever declared that the filesystem abstractions would not be merged; the discussion is all about the details of how that will happen. 

Ted Ts'o said that the Rust developers have been trying to avoid scaring kernel maintainers, and have been saying that "all you need is to learn a little Rust". But a little Rust is not enough to understand filesystem abstractions, which have to deal with that subsystem's complex locking rules. There is a need for documentation and tutorials on how to write filesystem code in idiomatic Rust. He said that he has a lot to learn; he is willing to do that, but needs help on _what_ to learn. (See [this article](<https://lwn.net/Articles/990619/>) for a discussion of how the Rust-for-Linux developers are working to meet this need). 

Torvalds said that it is not necessary to understand Rust to let it into a subsystem; after all, he said, _nobody_ understands the memory-management subsystem, but everybody is able to work with it. I pointed out that the Rust developers are not just creating subsystem bindings; they are trying to create inherently safe interfaces, and that often requires changes on the C side. That increases the impact on subsystems, but also makes the C code better. Airlie added that the Rust developers have to bring maintainers along with them, or the maintainers will not understand what is happening. 

Deacon raised the question of refactoring on the C side. Changing C interfaces will often have implications for the Rust code and may break it; somebody will the have to fix the problems. Torvalds said that, for now, breaking the Rust code is permissible, but that will change at some point in the future. Kroah-Hartman said that the Rust developers can take responsibility for the maintenance of the abstractions they add. 

#### Steam right ahead

Torvalds said that nothing depends on Rust in the kernel now, and nothing will for some time yet. What is important is to make forward progress, so developers should "steam right ahead" and not worry about these problems for now. It is enough to get things working, even though the details are not right. Once users are depending on Rust code, it will be necessary to worry more, he said, but kernel developers should not fail by being too careful now. 

Thomas Gleixner said that the Rust developers are careful about documenting their code, and he is not frightened by the prospect of refactoring it. If he does not understand something, he will simply send an email to the developer, just as he does with C code. Torvalds added that Rust has a lot to offer, and the kernel should try to take advantage of it. Kroah-Hartman said that it could eliminate entire classes of bugs in the kernel. 

Deacon asked how many developers are working on the Rust side now; Ojeda answered that there are currently six or seven people, most of whom are "real Rust experts". The strongest kernel expertise in the group had been Wedson Almeida Filho, who had recently [left the project](<https://lwn.net/Articles/987635/>). That was a real loss, but Ojeda is working to recruit others. 

Gleixner said that, 20 years ago, there had been a lot of fear and concern surrounding the realtime kernel work; he is seeing the same thing now with regard to Rust. We cannot let that fear drive things, he said. Torvalds said that Rust has been partially integrated for two years. That is nothing, he said; the project to build the kernel with Clang took a decade, and that was the same old language. 

Julia Lawall asked what happens when things change on the C side; how much will leak through into the Rust code? Bergmann said that reviewing Rust abstractions for a C subsystem without knowing Rust is not difficult; he can reach a point where he understands the code, but would not feel able to change it. 

Torvalds said that the community can play around with Rust for a few years. Gunthorpe, though, said that it would be good to get something into production; that would give the project some needed momentum. The binder driver might be a good choice. Ojeda said that would help to justify more support from companies. As the session closed, though, the primary outcome may well have been expressed by Torvalds, who suggested telling people that getting kernel Rust up to production levels will happen, but it will take years.
