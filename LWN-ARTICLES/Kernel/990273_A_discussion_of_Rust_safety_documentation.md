---
title: A discussion of Rust safety documentation
url: https://lwn.net/Articles/990273/
date: "September 17, 2024"
category: "Development tools-Rust"
author: "By Daroc Alden September 17, 2024 Kangrejos 2024"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Daroc Alden**  
September 17, 2024

* * *

[Kangrejos 2024](<https://lwn.net/Articles/990496/>)

[ Kangrejos 2024](<https://kangrejos.com/>) started off with a talk from Benno Lossin about his [recent work](<https://lwn.net/Articles/982868/>) to establish a standard for safety documentation in Rust kernel code. Lossin began his talk by giving a brief review of what safety documentation is, and why it's needed, before moving on to the current status of his work. Safety documentation is easier to read and write when there's a shared vocabulary for discussing common requirements; Lossin wants to establish that shared vocabulary for Rust code in the Linux kernel. 

Safety documentation has two parts, Lossin explained: requirements and justifications. Requirements are comments attached to functions that have been marked unsafe, and explain what must be true for the function to be used. He gave the example of the [ `Arc::into_raw()`](<https://doc.rust-lang.org/std/sync/struct.Arc.html#method.into_raw>) and [ `Arc::from_raw()`](<https://doc.rust-lang.org/std/sync/struct.Arc.html#method.from_raw>) functions that convert between a reference-counted smart pointer (`Arc`) and a plain pointer. For example, `from_raw()` must be called once for each call to `into_raw()` on a given allocation, otherwise the reference count will be incorrect. Also, `from_raw()` must be given a pointer that really did come from `into_raw()`, or it will do bad things to whatever object is being pointed to when the `Arc` is dropped and the reference count is decremented. 

[ ![\[Benno
Lossin\]](https://static.lwn.net/images/2024/benno-lossin-small.png) ](<https://lwn.net/Articles/990484>)

Those requirements should be spelled out in a safety comment attached to the function, Lossin said, so that users of the function know how to use it correctly. Furthermore, uses of that function should also have the second kind of safety documentation: justifications. Having a comment explaining _why_ the functions requirements hold at that particular call site makes it easier for reviewers to check whether the author of a patch really got their logic right, which prevents mistakes. 

Lossin briefly addressed the objection that these kinds of comments are not traditional in kernel C code. He said that Rust has "higher stakes" — since Rust's safety is its primary reason to exist, the Rust-for-Linux folks had better make it easy to write correct Rust code. Also, Rust has some more complex language features, such as smart pointers and compile-time lifetime tracking, that can be harder to work with when writing low-level unsafe code. Finally, actually writing out one's assumptions can sometimes catch errors that would otherwise have gone unnoticed. 

Once everyone had been brought up to date on what Lossin meant by safety documentation, he talked about the current status of his effort to document standards for safety documentation. He pointed out that safety documentation has actually been required by reviewers since the beginning; what he's asking for is to standardize the wording and formatting to make those comments easier to understand. 

Almost all existing unsafe blocks in the kernel have associated documentation. Soon, the project will enable a lint to warn about undocumented unsafe blocks. This is good, Lossin said, but there's a catch: the comments do not always use the same terminology, which can make it difficult to know whether they're correct. 

For example, some comments say that their function needs a "valid pointer" and some say that it needs a "valid, non-null pointer". But all valid pointers in Rust _are_ non-null. So is this just two ways of saying the same thing? Or is one of the comments using "valid" in a way that's not consistent with Rust's language documentation? It's impossible to tell what the actual requirements are without reading the code, which rather defeats the point of having concise safety documentation to read, Lossin said. 

Ideally, all of the comments would be correct, complete, and easy to understand. That's easier to accomplish if there's a shared vocabulary for common conventions — an author shouldn't need to write "valid, non-null" when just "valid" will do. Lossin suggested that they might want to standardize a dictionary of common terms, so that authors can write as little as possible, but readers will still be able to understand. Plus, having an explicit resource saying how to read safety documentation will make it easier for learners to come up to speed, and reduce the chances of misunderstandings between maintainers. 

Lossin ended the introductory portion of his talk by calling for people to read [ his RFC](<https://lwn.net/ml/all/20240717221133.459589-1-benno.lossin@proton.me/>), and get on the [ Rust-for-Linux project's Zulip chat server](<https://rust-for-linux.zulipchat.com/>) to talk about it. Then, he opened things up for a discussion. 

Richard Weinberger started by asking whether the safety documentation was meant to be human-readable or machine-readable. Lossin thought that was a good question for discussion, and tossed it back to the attendees. Andreas Hindborg, one of the organizers of Kangrejos, said "I like it when the documentation is human-readable". 

Daniel Almeida agreed, asking why the project would want the documentation to be machine-readable. Lossin suggested that it might be possible to have tooling process it, for formal verification. Almeida objected, saying that the existing code linter can ensure the comments are present, and what more is needed? 

Lossin suggested that perhaps they could write tooling to check the comments for correctness. Paul McKenney pointed out that if it were both human- and machine-readable, there could be tooling that would expand terse comments into a more detailed form using the dictionary Lossin had composed. 

Other attendees remained skeptical. One person pointed out that unsafe code is used exactly when Rust cannot statically guarantee something is safe — so trying to run static analysis on those parts sounds like a fool's errand. Lossin objected that they could simplify common checks, such as checking that a pointer is valid, or check kernel-specific invariants that the wider Rust world doesn't care about. He also pointed out that the Rust compiler has macros that can be used for annotations — those could be used to attach checks to different functions. The downside is that it would raise the difficulty of writing kernel Rust code, something that he was worried about. 

McKenney thought that even "stupid" formal verification can be helpful for catching mistakes. Even something as simple as checking that the requirement and justification comments match, somehow, would be valuable. 

Weinberger pointed out that the kernel already has [sparse](<https://www.kernel.org/doc/html/latest/dev-tools/sparse.html>), which works on C code, and that could potentially be expanded to Rust. Hindborg noted that there are people using formal verification on some drivers, such as checking that there are no writes outside of a DMA buffer. Formally verifying Rust code has proved to be easier than formally verifying C code in the past. 

Despite the possibility of improved tooling, Hindborg was against using machine-readable comments, saying that the Rust-for-Linux project is already bringing in a new language that's hard to accept. Adding some kind of formal notation on top of that will not go well. 

Lossin noted that you could make English machine-readable by restricting the grammar and vocabulary. He also pointed out that there are existing tools for formal verification. Despite that, he thought that it made more sense to "surgically insert" formal verification only where it would matter the most, making it opt-in. 

Almeida thought that even Lossin's idea of having a standardized dictionary of terms might be a step too far, saying that it could make it harder for people to contribute. Lossin suggested letting contributors know that they can ask for help when writing the comments during the review process if they have trouble with the dictionary. 

Miguel Ojeda raised the idea of having different standards for core and leaf functions — the Rust-for-Linux developers could set an example with the core APIs, and let that propagate. Lossin said that the normal documentation already works like that; the `kernel` crate is entirely documented (as is the Rust standard library), but it's up to individual subsystem maintainers to pick a standard that works for them. Since, ideally, most drivers will not need to use unsafe Rust, maybe the safety documentation is only needed for the `kernel` crate. 

José Marchesi asked how much of the information Lossin wanted comments to capture could instead be expressed in Rust itself. Lossin asked whether he meant in the type system or at run time, and Marchesi clarified that he meant at run time. In Ada, he said, there can be preconditions to a function, and he wondered if Rust had anything similar. 

Lossin agreed that it could be a good option to add Rust macros for that, if it could be made to work. Ojeda noted that some formal-verification people are already working on this, and may eventually get pre- and post-conditions into the language itself. But he didn't think that obviated the need for comments. 

Gary Guo raised the topic of foreign functions — sometimes, a function is unsafe to call only because it is a foreign function, and there aren't really any requirements to satisfy before using it. Lossin thought that the real problem there was the lack of documentation of the kernel's C functions. In order to know whether the FFI function is really safe, you need to look at the C code. He suggested that perhaps it could make sense to document foreign functions in the C code, and have [ bindgen](<https://docs.rs/bindgen/latest/bindgen/>) (the C-to-Rust bindings generator) transfer the documentation. Guo said that there were a lot of functions that do not really require documentation, and that documenting things in multiple places would certainly not be helpful. 

Maciej Falkowski asked why it was necessary to have obvious code accompanied by a safety comment. Lossin said that this gets into why safety documentation is needed in the first place. In C, the programmer needs to argue that the whole program is correct. Having unsafe blocks reduces that global problem to a local property, Lossin said. So if the unsafe blocks are small, only a small amount of context is needed to make sure it's right — which makes the whole system more correct. From that point of view, safety documentation is actually _better_ if it's obvious, because that lets maintainers and people working on the code later have more confidence that the code is correct. 

Overall, the attendees were mostly sympathetic to the need for standardized safety documentation, although there was clearly still some disagreement around exactly what form that should take, and whether it should be used as the basis for new tooling. Eventually, the session ran out of time. 

[ Thanks to the Linux Foundation, LWN's travel sponsor, for supporting our coverage of Kangrejos. ]
