---
title: Rust for filesystems
url: https://lwn.net/Articles/978738/
date: "June 21, 2024"
category: "Development tools-Rust; Filesystems-Internal API"
author: "By Jake Edge June 21, 2024 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
June 21, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

At the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Wedson Almeida Filho and Kent Overstreet led a combined storage and filesystem session on using Rust for Linux filesystems. Back in December 2023, Almeida had [posted an RFC patch set](<https://lwn.net/ml/linux-fsdevel/20231018122518.128049-1-wedsonaf@gmail.com/>) with some Rust abstractions for filesystems, which resulted in [some disagreement over the approach](<https://lwn.net/Articles/958072/>). On the same mid-May day as the session, he [posted](<https://lwn.net/ml/all/20240514131711.379322-1-wedsonaf@gmail.com/>) a second version of the RFC patches, which he wanted to discuss along with other Rust-related topics. 

#### Goals

After updating attendees on the status of his patches, Almeida listed some of the goals of the [Rust-for-Linux project](<https://rust-for-linux.com/>), which are embodied in the filesystem abstractions that he is proposing. The first is to express more of the requirements using Rust's type system in order to catch more mistakes at compile time. In addition, the project's developers want to automate some tasks, such as cleaning up resources, in ways that are not easily available to C code. The overall idea is to have a more productive filesystem-development experience, with less time spent on debugging problems that the compiler could find, and with fewer memory-related vulnerabilities overall. 

[ ![\[Wedson Almeida Filho\]](https://static.lwn.net/images/2024/lsfmb-almeida-sm.png) ](<https://lwn.net/Articles/979161/>)

Overstreet said that he had been a part of too many two-week bug hunts and has been trying to find ways to avoid those kinds of problems for bcachefs. The Rust language provides a lot more than what he can do in C; it eliminates undefined behavior and provides facilities to see what is happening inside the code. "You can't debug, if you can't see what's going on." He believes that kernel development "will get a whole lot easier over the coming decades" due to using Rust. It will be possible to prove the correctness of code written in Rust, which will mean that bugs that can derail feature development will be much less common. 

From his [slides](<https://docs.google.com/presentation/d/1dflxtAojRu3mdx19oBHP3vXCXI1vocFIVc5jVaf9FLA/>), Almeida showed an example of how the Rust type system can eliminate certain kinds of errors. He noted that the [`iget_locked()`](<https://elixir.bootlin.com/linux/v6.9.5/source/fs/inode.c#L1248>) function in current kernels has a complicated set of requirements. Callers must check to see if the return value is null and, if it is not, then the contents of the returned [`struct inode`](<https://elixir.bootlin.com/linux/v6.9.5/source/include/linux/fs.h#L632>) need to be checked to see if it is a new or existing inode. If it is new, it needs to be initialized before it can be used; if that fails, [`iget_failed()`](<https://elixir.bootlin.com/linux/v6.9.5/source/fs/bad_inode.c#L244>) needs to be called, he said. 

There was some discussion of the finer points of what callers of `iget_locked()` need to do, with Al Viro disagreeing with some of what Almeida had on his slide. That went back and forth, with Overstreet observing that it was exactly that kind of discussion/argument that could be avoided by encapsulating the rules into the Rust types and abstractions; the compiler will know the right thing to do. 

Overstreet noted that Christian Brauner and Alice Ryhl have helped to improve the abstractions a great deal since the first posting; in particular, there are things he has learned about reference counts based on how they are being handled by the Rust code. "This is going to make all our lives so much easier", Overstreet said. 

Almeida put up a slide with the equivalent of `iget_locked()` in Rust, which was called `get_or_create_inode()`. The important part is the return type, he said; as with C, callers must check for failure, but the success case is much different. If it is successful, the caller either receives a regular reference-counted inode to use (which has its reference count automatically decremented when the inode object is no longer referenced) or it receives a new inode, which will automatically call the equivalent of `iget_failed()` if it is never initialized. If it is ever initialized (which can only be done once), it becomes a regular inode with the automatic reference-count decrement. All of that is enforced through the type system. 

Viro seemed somewhat skeptical of how that would work in practice. He wondered where in the source code those constraints would be defined. Almeida said that the whole idea is to determine what the constraints are from Viro and other filesystem developers, then to create types and abstractions that can enforce them. 

#### Disconnect

Dave Chinner asked about the disconnect between the names in the C API and the Rust API, which means that developers cannot look at the C code and know what the equivalent Rust call would be. He said that the same names should be used or it would all be completely unfamiliar to the existing development community. In addition, when the C code changes, the Rust code needs to follow along, but who is going to do that work? Almeida agreed that it was something that needs to be discussed. 

As far as the renamed functions goes, he is not opposed to switching the names to match the C API, but does not think `iget_locked()` is a particularly good name. It might make sense to take the opportunity to create better names. 

There was some more discussion of the example, with Viro saying that it was not a good choice because `iget_locked()` is a library function, rather than a member function of the superblock object. Almeida said that there was no reason `get_or_create_inode()` could not be turned into a library function; his example was simply meant to show how the constraints could be encoded in the types. 

Brauner said that there needs to be a decision on whether the Rust abstractions are going to be general-purpose, intended for all kernel filesystems, or if they will only be focused on the functionality needed for the simpler filesystems that have been written in Rust. There is also a longer-term problem in handling situations where functions like `get_or_create_inode()` encode a lot more of the constraints than `iget_locked()` does. As the C code evolves, which will happen more quickly than with the Rust code, at least initially, there will be a need to keep the two APIs in sync. 

It comes down to a question of whether refactoring and cleanup will be done as part of adding the Rust abstractions, Overstreet said; he strongly believes that is required. But there is more to it than just that, James Bottomley said. The object lifecycles are being encoded into the Rust API, but there is no equivalent of that in C; if someone changes the lifecycle of the object on one side, the other will have bugs. 

There are also problems because the lifecycle of inode objects is sometimes filesystem-specific, Chinner said. Encoding a single lifecycle understanding into the API means that its functions will not work for some filesystems. Overstreet said that filesystems which are not using the VFS API would simply not benefit, but Chinner said that a VFS inode is just a structure and it is up to filesystems to manage its lifetime. Almeida said that the example would only be used by filesystems that currently call `iget_locked()` and could benefit. The Rust developers are not trying to force filesystems to change how they are doing things. 

#### Allocating pain

Part of the problem, Ted Ts'o said, is that there is an effort to get "everyone to switch over to the religion" of Rust; that will not happen, he said, because there are 50+ different filesystems in Linux that will not be instantaneously converted. The C code will continue to be improved and if that breaks the Rust bindings, it will break the filesystems that depend on them. For the foreseeable future, the Rust bindings are a second-class citizen, he said; broken Rust bindings are a problem for the Rust-for-Linux developers and not the filesystem community at large. 

He suggested that the development of the Rust bindings continue, while the C code continues to evolve. As those changes occur, "we will find out whether or not this concept of encoding huge amounts of semantics into the type system is a good thing or a bad thing". In a year or two, he thinks the answer to that will become clear; really, though, it will come down to a question of "where does the pain get allocated". In his mind, large-scale changes like this almost always come down to a "pain-allocation question". 

Almeida said that he is not trying to keep the C API static; his goal is to get the filesystem developers to explain the semantics of the API so that they can be encoded into Rust. Bottomley said that as more of those semantics get encoded into the bindings, they will become more fragile from a synchronization standpoint. Several disagreed with that, in the form of a jumble of "no" replies and the like. Almeida said that it was the same with any user of an API; if the API changes, the users need to be updated. But Ts'o pointedly said that not everyone will learn Rust; if he makes a change, he will fix all of the affected C code, but, "because I don't know Rust, I am not going to fix the Rust bindings, sorry". 

Viro came back to his objections about the proposed replacement for `iget_locked()`. The underlying problem that he sees is the reliance on methods versus functions; using methods is not the proper way forward because the arguments are not specified explicitly. But Overstreet said that the complaints about methods come from languages like C++ that rely too heavily on inheritance, which is "a crap idea". Rust does not do so; methods in Rust are largely just a syntactical element. 

There was some discussion of what exactly is being encoded in the types. Jan Kara said that there is some behavior that goes with the inode, such as its reference count and its handling, but there is other behavior that is inherent in the `iget_locked()` function. Overstreet and Almeida said that those two pieces were both encoded into the types, but separately; other functions using the inode type could have return values with different properties. 

Viro went through some of his reasoning about why inodes work the way they do in the VFS. He agreed with the idea of starting small to see where things lead. Overstreet suggested that maybe the example used was not a good starting point, "because this is the complicated case". "Oh, no it isn't", Viro replied to laughter as the session concluded.
