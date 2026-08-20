---
title: The perils of pinning
url: https://lwn.net/Articles/907876/
date: "September 15, 2022"
category: "Development tools-Rust"
author: "By Jonathan Corbet September 15, 2022 Kangrejos"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 15, 2022

* * *

[Kangrejos](<https://lwn.net/Archives/ConferenceByYear/#2022-Kangrejos>)

Parts of the Rust language may look familiar to C programmers, but the two languages differ in fundamental ways. One difference that turns out to be problematic for kernel programming is the stability of data in memory — or the lack thereof. A challenging session at the [2022 Kangrejos](<https://kangrejos.com>) conference wrestled with ways to deal with objects that should not be moved behind the programmer's back. 

C programmers take full responsibility for the allocation of memory and the placement of data structures in that memory. Rust, instead, takes most of that work — and the associated control — out of the programmer's hands. There are a number of interesting behaviors that result from this control, one of which being that the Rust compiler will happily move objects in memory whenever that seems like the thing to do. Since the compiler knows where the references to an object are, it can move that object safely — most of the time. 

Things can go badly, though, when dealing with self-referential data structures. Consider the humble (C) `list_head` structure that is heavily used in the kernel: 

```
struct list_head {
    	struct list_head *next, *prev;
        };
```

It is possible to create a Rust wrapper around a type like this, but there are complications. As an example, initializing a `list_head` structure to indicate an empty list is done by setting both the `next` and `prev` fields to point to the structure itself. If, after that happens, the compiler decides to move the structure, those pointers will now point to the wrong place; the resulting disorder does not demonstrate the sort of memory safety that Rust hopes to provide. The same thing can happen with non-empty lists, of course, and with a number of other kernel data structures. 

Benno Lossin started his Kangrejos talk by saying that Rust provides a mechanism to deal with this problem, which [can also arise](<https://blog.cloudflare.com/pin-and-unpin-in-rust/>) in pure Rust code. That mechanism is the [`Pin`](<https://doc.rust-lang.org/std/pin/index.html>) wrapper type; placing an object of some other type into a `Pin` will nail down its location in memory so that it can no longer be moved. There are numerous complications, including the need to use `unsafe` code to access fields within a pinned structure and to implement "pin projection" to make those fields available generally. 

[![\[Benno Lossin\]](https://static.lwn.net/images/conf/2022/kang/BennoLossin-sm.png)](<https://lwn.net/Articles/907891/>) The really big challenge, though, is in a surprising area: initialization. Fully understanding the issues involved requires a level of Rust guru status far beyond anything your editor could hope to attain, but it seems to come down to a couple of aspects of how Rust treats objects. Rust goes out of its way to ensure that, if an object exists, it has been properly initialized. Object initialization in Rust tends to happen on the stack, but objects that need to live indefinitely will need to move to the heap before being pinned. That movement will break a self-referential object, but pinning before initialization will break Rust's memory-safety rules. 

Solutions exist, but require a lot of `unsafe` code; Lossin has been working on alternatives. He initially tried to use const generics to track initialization, but the solution required the use of [procedural macros](<https://doc.rust-lang.org/reference/procedural-macros.html>) and was complex overall. And, at the end, it was "unsound", a Rust-community term indicating that it was not able to properly handle all cases. So that approach was abandoned. 

Instead, he has come up with a solution that uses (or abuses) struct initialization and macros. Your editor will not attempt a full description of how it works; the whole thing can be seen in [Lossin's slides](<https://kangrejos.com/Safe%20Pinned%20Initialization%20in%20Rust.pdf>). Among other things, it requires using some complex macros that implement a not-Rust-like syntax, making the code look foreign even to those who are accustomed to Rust. 

The response in the room was that, while this work is clearly a clever hack, it looks like a workaround for a limitation of the Rust language. It's the kind of thing that can create resistance within the kernel community, many members of which already find Rust hard to read (though it should be said that kernel developers are entirely willing to merge C preprocessor hackery when it gets the job done). There was a strong desire to see a different solution. 

Xuan "Gary" Guo stepped up to show an alternative approach. In C, he began, it is easy to create an object without initializing it, or to initialize an object twice. In Rust, anything that circumvents the normal initialization routine requires `unsafe` code. Tracking the initialization of such objects can require maintaining an initial variable to hold the current state, which would be a good thing to avoid. Other approaches can require additional memory allocations, which are also not good for the kernel. 

[![\[Xuan Guo\]](https://static.lwn.net/images/conf/2022/kang/XuanGuo-sm.png)](<https://lwn.net/Articles/907893/>) There have been attempts to address the problem with, for example, the [`pin_init` crate](<https://docs.rs/pin-init/latest/pin_init/>). But pin_init still is unable to initialize self-referential structures, and has to do its own parsing of Rust structures and expressions. That requires the [`syn` crate](<https://docs.rs/syn/latest/syn/>), which is not really suitable for kernel building. 

A proper solution for the kernel, he said, would have a number of characteristics. It should be safe, impose no extra cost, and require no additional memory allocations. Aggregation should work; a structure containing multiple pinned objects should initialize properly. The mechanism used should not look much different from normal Rust. There should also be no assumptions about whether initialization can fail or not. 

Guo's solution (which can be seen in [his slides](<https://kangrejos.com/pin-init%20-%20safe%20initialisation%20of%20pinned%20structs.pdf>)), looks a bit closer to normal Rust than Lossin's, but it still depends on complex macro trickery and has not managed to avoid using the `syn` crate. And it still can't handle self-referential structures properly. But it is arguably a step in the right direction. 

Once again, though, the proposed solution looked like an impressive hack, but the response was not entirely favorable. Kent Overstreet described it as "really gross", adding that this job should be done by the compiler. Wedson Almeida Filho responded that the compiler developers would just suggest using procedural macros instead. One compiler developer, Josh Triplett, happened to be in the room; he said that there could be help provided by the language, but that requires an RFC describing the desired behavior, and nobody has written it yet. The session wound down without any specific conclusions other than, perhaps, a desire to pursue a better solution within the Rust language rather than trying to work around it. 

[Thanks to LWN subscribers for supporting my travel to this event.]
