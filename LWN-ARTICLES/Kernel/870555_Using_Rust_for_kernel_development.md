---
title: Using Rust for kernel development
url: https://lwn.net/Articles/870555/
date: "September 27, 2021"
category: "Development tools-Rust"
author: "By Jonathan Corbet September 27, 2021 Maintainers summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 27, 2021

* * *

[Maintainers summit](<https://lwn.net/Articles/870415/>)

The Rust for Linux developers were all over the [2021 Linux Plumbers Conference](<https://linuxplumbersconf.org/>) and had many fruitful discussions there. At the Maintainers Summit, Miguel Ojeda stepped away from Plumbers to talk about Rust in a different setting. What will it take to get the Rust patches merged? The answers he got were encouraging, even if not fully committal. 

Ojeda started by asking the group whether the community wanted Rust in the kernel. If it goes in, he said, it should do so as a first-class citizen. In his discussions he has encountered a number of kernel developers who are interested in the language; many of them are quite open to it. He has gotten help from a number of those developers in the process. Some groups, including the Android team, actively want it, he said. 

Linus Torvalds answered that the kernel community will "almost certainly" do a trial with the language, but that the Rust developers need to accept that it's a trial. It is not necessary to convince everybody in the kernel community before this can happen, but there does need to be a certain level of buy-in from the subsystem maintainers who will be directly affected at the beginning. The support from "fake Linus" (GPIO maintainer Linus Walleij) and Greg Kroah-Hartman is a good start, he said; a majority of kernel maintainers is not mandatory. Torvalds had looked at the patches a few months ago when they were posted, and nothing therein made him say "no way". He has not seen any postings since, though. If Rust support is not merged, Torvalds ended, it will never get to be good enough for real use in the kernel. 

Kroah-Hartman said that the Rust patches are looking a lot better, but aren't ready yet. The [Rust GPIO driver](<https://lwn.net/Articles/863459/>) that Wedson Almeida Filho posted in July was "awesome", and he added that a number of filesystem developers are interested in Rust. That could be a good place to work, since the virtual filesystem APIs in the kernel are relatively stable. 

Kees Cook suggested that WiFi or Bluetooth drivers could be a good place to use Rust; Kroah-Hartman answered that he would love to drop all of the current Bluetooth drivers. He said (to groans from the group) that he knows of an upcoming phone that is shipping with 100 out-of-tree drivers; he suggested that developers interested in Rust pick ten of those and see how things work. 

Dave Airlie said that some maintainers will certainly be scared by the addition of a new language; they are going to have to take the time to learn it by writing something useful. They will need to know that there are resources out there to help them in dealing with Rust code in their subsystems. Torvalds said that Rust is not that hard to read, even if the error-handling patterns are very different. Anybody who can review patches should be able to pick up enough Rust to review code in that language. 

Ted Ts'o suggested that the Rust developers should post patches more regularly — every week or two. Developers will look when something shows up in their inbox, that is the way to get their attention. 

Airlie said that there are examples of Rust code at the edges of the kernel, such as drivers. Has any work gone into putting Rust into the core, with C code at the edges? Ojeda answered that the Rust developers are not trying to rewrite things in the core kernel; instead, they are making a set of abstractions so that drivers can be written in safe rust. A C driver using a Rust core would lose a lot of the advantages of using Rust, he said; once you go to Rust, you want to stay there. 

Ts'o raised the often-heard concern of wide-ranging, cross-subsystem changes. How hard are those going to be when there is Rust code involved too? It is fine if the GPIO maintainer buys in, since that subsystem is reasonably well contained. But if, say, the filesystem developers have to make a change that breaks the Rust GPIO interface, what happens then? Ojeda repeated that kernel maintainers need to buy into the change, and that they need to be ready to review changes to their subsystem. Developers from other subsystems who want to make a change in a Rust-using subsystem will need to learn the language. The Rust developers can provide help, but it will not be enough; if the kernel maintainers want Rust, they are going to have to help. Mark Brown said that, even when maintainers are enthusiastic, review from the Rust developers to be sure that work is "tasteful" will be important. After all, he just started learning the language one week ago and doesn't know what he is doing at this point. 

Arnd Bergmann said that putting the Rust code into a corner of the kernel tree is not the right approach. In the current patches there is one top-level Rust directory, but it would be far better to distribute the Rust code among the affected subsystems. As little of that code as possible should be in a central location. Ojeda answered that there needs to be some general support code, but that a lot of things could be split out of the current kernel crate and put into subsystem-specific crates. 

Al Viro asked about the stability of the Rust toolchain, noting that the current requirement to use the latest Rust compiler is a problem. At least once a month he ends up bisecting a problem over three or four years of history; if a different compiler is needed at each bisection step, things are not going to work. Ojeda said that the kernel work is using unstable Rust features now, and that is a problem; it does make compatibility harder. For now, the Rust patches support a single Rust version for each kernel release, and they cannot guarantee that a newer compiler will work later on. So yes, bisection would require changing compiler versions. 

If, however, Rust support gets into the mainline kernel, the situation might change. That would put pressure on the Rust development community to stabilize the needed features in the language — though there is never a guarantee that this will actually happen. Sooner or later, though, it will be possible to build a kernel using only stable Rust features; at that point the compatibility problems should go away. He would understand, he said, if the community chose to not merge Rust support until that happens. 

Ts'o said that, if the Rust community wants the public-relations win that would (presumably) come from use in the kernel, it might well feel moved to stabilize the needed features. Ojeda said that he had been invited to Rustconf this year, which was a good sign, but then his submission on Rust in the kernel was rejected — a less-good sign. The consideration of Rust for the kernel was then highlighted at that conference, though, so there is definitely some interest in the Rust community at some level. 

Thomas Gleixner said that he is not opposed to the experiment, and he likes some of the concepts in the language. He is worried, though, about "the blank page where the memory model should be". Ojeda said that the Rust community is taking its memory-model cues from the C++11 and C11 standards, but nothing is finalized. During a conversation earlier that week, Ojeda said, he encouraged Paul McKenney that this is the best time to go to the Rust community and tell them how things could be; this is an opportunity to fix issues in the C memory model and do things right. 

Torvalds concluded the session by reiterating that bringing Rust into the kernel is an experiment, the community is just putting its toes into the water. It will take years to sort things out and determine whether it actually works in the kernel. He is positive toward Rust, he said, and likes the language; he wants a safer language, especially for driver development. But, he admonished at the end, he does not expect to rewrite the whole kernel in the Rust language.
