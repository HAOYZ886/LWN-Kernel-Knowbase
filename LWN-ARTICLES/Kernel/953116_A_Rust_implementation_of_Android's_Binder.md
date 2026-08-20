---
title: "A Rust implementation of Android's Binder"
url: https://lwn.net/Articles/953116/
date: "November 30, 2023"
category: "Android; Development tools-Rust"
author: "By Jonathan Corbet November 30, 2023 LPC"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 30, 2023

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2023-Linux_Plumbers_Conference>)

The Android system was once famous for extensive, out-of-tree kernel enhancements. Many of those have been eliminated or upstreamed over the years, bringing Android much closer to the mainline kernel. One significant component in the "upstreamed" category is Binder, an interprocess communication mechanism that is used only by Android. There are a number of factors that make Binder a good candidate for rewriting in the Rust language; at the [2023 Linux Plumbers Conference](<https://lpc.events/event/17/page/198-lpc-2023-overview>), Carlos Llamas and Alice Ryhl described the motivation behind and implementation of a rewrite of Binder in Rust. 

[![\[Carlos Llamas\]](https://static.lwn.net/images/conf/2023/lpc/CarlosLlamas-sm.png)](<https://lwn.net/Articles/953120>) Llamas began the talk by describing Binder as the primary interprocess communication channel for the Android system. It has a long history, having originated in [BeOS](<https://en.wikipedia.org/wiki/BeOS>) and been used in [Palm OS](<https://en.wikipedia.org/wiki/Palm_OS>) before being adopted by Android. The Binder driver lives in the kernel; most applications access it by way of the libbinder library. The Binder service implements a thread pool that is able to take requests from clients in parallel; a memory-mapped region is used for transactions between server and client processes. 

There are a number of Binder features that make it attractive for Android. It performs no buffering at all in the kernel. There is an extensive set of operations, all accessed via `ioctl()`, including a combined read/write command. Binder implements priority inheritance and the ability to share file descriptors; it has a number of remote object management features and a strong reference-counting implementation. 

Given Binder's long history, why would one want to rewrite it now? The module, Llamas said, features a high level of complexity and a large amount of technical debt; that combination has proved to be a recipe for security problems. Analysis has shown that Binder contains about 3.1 vulnerabilities per thousand lines of code, which is a relatively high number — and it is not getting better. Exploits are known for about half of the known vulnerabilities. Given that even the lowest-privilege sandboxes in Android have access to Binder, this security record is troubling indeed. 

There is a lot that can be improved in Binder, Llamas said. Functions nearly 1,000 lines long ([example](<https://elixir.bootlin.com/linux/v6.6.2/source/drivers/android/binder.c#L2896>)) are generally a bad sign. The error handling in Binder is "problematic" and the source of a lot of bugs. But any improvements to this complex body of code tend to be risky, leading to a lot of "fix to the fix to the fix" patches. The Android developers are tired of chasing bugs like these and are looking for a better way. 

#### The new Binder

Ryhl then took over to talk about the new Binder implementation, which was [posted to the mailing lists](<https://lwn.net/ml/linux-kernel/20231101-rust-binder-v1-0-08ba9197f637@google.com/>) at the beginning of November. One of the big advantages of working in Rust, Ryhl said, is that it makes object ownership visible in the type system. For example, C code working with reference-counted objects has to check for a non-zero reference count at run time, but Rust code simply will not compile if it lacks an object reference where one is needed. In C, one can look at a structure full of function pointers and get no sense for which of those manipulate reference counts; in Rust, that can be made clear by using different types. 

Rust also makes it impossible to forget to clean up in error paths. [The slides](<https://lpc.events/event/17/contributions/1427/attachments/1177/2423/rust_binder_plumbers2023.pdf>) for the talk pointed to [some error-handling code in the C implementation](<https://elixir.bootlin.com/linux/v6.6.2/source/drivers/android/binder.c#L3663>) that is implemented as a long series of `goto` targets at the end of a function; the Rust equivalent was simply the "`}`" that ends the function. Rust also actively prevents almost all of the vulnerabilities that have been found in Binder; over half of them were use-after-free bugs, which "never happen in Rust". Other types of vulnerabilities, such as out-of-bounds array accesses, will be caught at run time, turning a potential compromise into a denial-of-service problem — a much less severe situation. 

[![\[Alice Ryhl\]](https://static.lwn.net/images/conf/2023/lpc/AliceRhyl-sm.png)](<https://lwn.net/Articles/953122>) Ryhl has thus duly rewritten Binder in Rust. The result implements all of the features provided by the C implementation and passes the entire Binder test suite; it is able to boot and run an Android device. Ryhl brandished a phone that was running the Rust Binder implementation as proof, to applause. The performance is as good as the C version; it "took a bit of optimization" to reach that point, but the C version has also seen a fair amount of optimization work. 

Rust, she said, can be rather more verbose than C; there are a lot of invariants that have to be expressed in the code. But that is countered by the need for far less error-handling code; it turns out to be a wash, with the size of the two implementations being about the same. 

With regard to unsafe code, she said that there will always be a need for some unsafe code in the kernel, but that the Binder implementation needed little of it, mostly to interface with the (still in C) binderfs implementation. The abstractions needed to interface with the rest of the kernel can involve unsafe code in general; the workqueue abstractions (which Ryhl [upstreamed for 6.7](<https://git.kernel.org/linus/15b286d1fd05>)) have a fair amount of it. But that is a single module that will be shared across all drivers using workqueues; it only has to be gotten right once. 

Writing in Rust does not guarantee the end of vulnerabilities, she said at the conclusion of the talk; one was discovered in the Rust implementation during the conference. It turned out that the C implementation contains a similar bug. In C, that bug can be exploited to cause a use-after-free vulnerability, which is "game over". In Rust, there is no use-after-free, but the result could be a confusion of the mappings used to map messages, which is still a severe vulnerability, though rather harder to exploit. Using Rust, she concluded, prevented the memory-safety problem, but logic errors remain. 

In the questions after the talk, Julia Lawall asked about the kinds of optimizations that were needed for Rust code. A source of bottlenecks, Ryhl said, is calling into C to access tiny functions. Performance can be improved by rewriting those functions in Rust, but that leads to duplicated code and the associated maintainability problems. 

There were a couple of questions about abstractions and their relationship to the C code. One attendee asked if, during the process of implementing Binder, changes in the kernel's C code had forced changes to the abstractions. The answer was that there was one change in process freezing that has caused some problems. I then asked about the workqueue abstractions; if somebody makes an incompatible change to the (C) workqueue implementation, who is responsible for making the abstractions work again? The answer to that is not entirely clear; eventually, it seems, the kernel's usual "you broke it, you fix it" rule will probably apply, but it is expected that many developers will lack the Rust skills needed to update those abstractions for some time and will thus need help from the Rust-for-Linux developers. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting our travel to this event.]
