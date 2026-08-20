---
title: More Rust concepts for the kernel
url: https://lwn.net/Articles/869428/
date: "September 20, 2021"
category: "Development tools-Rust"
author: "By Jonathan Corbet September 20, 2021 Kangrejos"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 20, 2021

* * *

[Kangrejos](<https://lwn.net/Archives/ConferenceByYear/#2021-Kangrejos>)

The [first day](<https://lwn.net/Articles/869145/>) of the [Kangrejos](<https://kangrejos.com/>) (Rust for Linux) conference introduced the project and what it was trying to accomplish; [day 2](<https://lwn.net/Articles/869317/>) covered a number of core Rust concepts and their relevance to the kernel. On the third and final day of the conference, Wedson Almeida Filho delved deeper into how Rust can be made to work in the Linux kernel, covered some of the lessons that have been learned so far, and discussed next steps with a number of kernel developers. 

Almeida started by noting that he is not a Rust developer and does not feel that the language is perfect; he does believe, though, that it can solve some problems in the kernel. He works as an Android platform security-team engineer and has been looking for ways to improve that platform — specifically, to reduce its attack surface. Rust can do that, he said; it also helps with correctness and provides an expressive type system with features that C cannot match. 

#### Ownership

[![\[Wedson Almeida
Filho\]](https://static.lwn.net/images/conf/2021/rust/WedsonAlmeidaFilho-sm.png)](<https://lwn.net/Articles/869625/>) An important Rust concept, he continued, is data ownership; every value in a Rust program has exactly one owner. That ownership will move around as the program executes, but will never be shared. When the owner of an object goes out of scope, that object will be freed. Ownership is exclusive, but there are still ways to move data around, of course; in Rust that is done with references. A mutable reference allows the holder to modify the data; mutable references are exclusive, in that no other references to the object in question may exist. Shared references, instead, are non-exclusive, and are read-only. 

There are, it seems, a couple of exceptions to the above rules. Almeida mentioned briefly the idea of [interior mutability](<https://doc.rust-lang.org/book/ch15-05-interior-mutability.html>) but did not get into the details of it. Rust also supports raw pointers, but only in unsafe code. 

One result of Rust's ownership rules is that there are no data races in code written in safe Rust. To have a data race, you must have at least two CPUs accessing shared data in an unsynchronized manner, with at least one writer. Since mutable references are exclusive, this situation simply cannot come about. 

These rules also give the compiler the ability to optimize around code that it cannot directly see. For example, in code that looks like: 

```
*x = 32
        some_function()
        return *x
```

The compiler can safely just return 32. There cannot be aliases for the variable `x`, so there is no way that `some_function()` can have modified its value. 

So how do the ownership rules apply to kernel code? Almeida brought up the example of the `private_data` field found in many kernel structures; it allows subsystems to attach their own data to a structure managed by higher levels of the system. That field is typically a `void *` pointer that is cast to the appropriate type when it is used; this is not safe usage by Rust standards. As an example, consider [`struct file`](<https://elixir.bootlin.com/linux/latest/source/include/linux/fs.h#L920>), which is used in the kernel to represent an open file. 

In Rust code, a developer would write an `open()` method that creates some sort of internal state — the sort of data that would normally be attached via `private_data`. That state will be returned to the caller, and ownership of that state object is returned as well. If, for example, user space calls `ioctl()` on that open file, the handler will get a shared reference to the state object; this must be shared, since these calls can be made concurrently. When, instead, the file is closed, the `release()` method will get ownership of the state, which it will then drop. All of this can be done using Rust's type discipline to provide type and concurrency safety. 

#### Device IDs, locks, and more

Another example is the device-ID tables found throughout the driver subsystem. These tables contain an array of IDs and an untyped, optional argument; they must be null-terminated. Almeida has written a [PL061 (GPIO) driver](<https://lwn.net/Articles/863459/>) in Rust to mirror [the C driver](<https://elixir.bootlin.com/linux/latest/source/drivers/gpio/gpio-pl061.c>) found in the kernel and created a new device-ID-table abstraction to go with it. Developers must decide what the type of the optional data will be, and all data specified must match that type. There is no need to null-terminate the list, eliminating a common error made by driver authors. As a result, the whole mechanism is type-safe and easier to use. 

Locking is the cause of a lot of complexity and confusion in the kernel. In C code, locks are typically declared as structure fields, and it is often not clear exactly what data any given lock is protecting. In Rust, instead, data is associated directly with the lock that protects it, with the result that there is no way to write code that accesses that data without first acquiring the lock — at least, not if the developer wants the compiler to accept it. All of the checking can be done at compile time. 

Almeida briefly mentioned the vulnerability known as [CVE-2021-26708](<https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-26708>), which was an exploitable race condition in the mainline kernel caused by accessing data prior to acquiring the lock that protects it. Rust, he said, would have prevented this vulnerability from happening. The definition of the lock would have ensured that the relevant data could not be touched without first acquiring that lock. But what happens if a developer doesn't realize that a lock is needed in the first place? In this case Rust's ownership rules would save the situation, since attempts to modify the unprotected data would fail with the wrong type of reference. 

Laurent Pinchart jumped in to say that he likes the idea of bundling data with the lock that protects it. But he wondered about the cases where the developer knows that the lock is not needed; initialization code is one such case. Seemingly, if the compiler knows that there can only be one reference to the object in question, the lock is not necessary; this can happen when the object has just been created, for example. If all else fails, Almeida said, developers can always use the "unsafe escape hatch". 

Another abstraction to look at is file descriptors; these are created by first acquiring a reference to the underlying `file` structure, then allocating the descriptor number itself. If that allocation fails, code must remember to drop the file reference. Rust's lifecycle management makes that error-handling happen automatically, with a lot less boilerplate code. The sort of "`goto out;`" pattern often seen in kernel code is just not necessary in Rust. 

[CVE-2019-15971](<https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-15791>), Almeida said, came about as the result of a failure to increment a file's reference count; that led to a use-after-free vulnerability. In Rust it is not possible to make this kind of mistake without getting a friendly diagnostic from the compiler. 

Returning to error paths, Almeida repeated that most of the complex and error-prone error handling seen in kernel code just vanishes when using Rust. For the most part, objects are simply cleaned up when they go out of scope, with no explicit handling needed at all. For cases where the developer needs more control, a [`scopeguard`](<https://docs.rs/scopeguard/1.1.0/scopeguard/>) object can be used. This object is initialized with the appropriate error-handling information; if all goes well and the error-handling code is not needed, its `dismiss()` method can be called. Otherwise, if that object goes out of scope, it will perform the needed error handling. 

A few other kernel abstractions were covered, including task structures, red-black trees, and access to memory-mapped I/O areas. The point that Almeida was trying to make should be clear by now, though. The Rust language is capable of handling kernel-level programming in a way that is inherently safer than doing the same tasks in C. Were there, he concluded, any questions or objections? 

#### Discussion

Julia Lawall started by asking about things that _don't_ work well in Rust — where are the sticking points? Almeida answered that, in Rust, all objects are movable in memory, which can be problematic for self-referential data structures. The solution is [pinning](<https://doc.rust-lang.org/std/pin/index.html>), but that requires writing unsafe code. A lot of unsafe code in existing Rust drivers has its origin in this problem, he said. Another issue is the way that Rust requires all data to be initialized, which can be a problem with mutexes in particular. Fixing that is mostly a matter of finding the right abstractions. 

Pinchart questioned the incremental approach being taken by the Rust developers — an issue that came up a number of times in the discussion. He said that the right approach is to lock the kernel subsystem maintainers in a room, require them to learn Rust, and to provide a great deal of help [![\[Discussion\]](https://static.lwn.net/images/conf/2021/rust/day3-disc-sm.png)](<https://lwn.net/Articles/869626/>) to those maintainers during the transition. If a maintainer is blocked by a Rust-related problem, they should be able to count on immediate help to fix it. Otherwise, he said, the Rust developers are going to encounter a huge amount of pushback. 

Ojeda questioned whether pushback was really going to be a problem; nobody is taking away the ability to write a driver in C, he said. Pinchart replied that, at some point, maintainers may not accept those drivers. Ojeda said that could maybe happen in five or ten years, not before. It is necessary to say that kernel developers will be expected to learn some Rust, he said; the safe subset is not that hard. He did allow that unsafe Rust is rather harder; worries about undefined behavior return, and the documentation is not as good. 

In another frequently heard concern, Pinchart pointed out that kernel developers have to be able to work all over the source tree; it's a rare developer who never looks outside of one subsystem. That will make it hard to minimize the impact of Rust in the kernel early on; a surprising number of developers are likely to encounter it. Ojeda said that the plan is only to introduce Rust in subsystems where the maintainer knows the language, but Pinchart answered that expecting that other developers will not have to care about that Rust code is wrong. Developers are almost certainly going to have to work with unsafe Rust as well. Mark Brown added that there will probably have to be some sort of flag day tied to the introduction of Rust, after which maintainers will simply have to know the language to some degree. 

Almeida asked how many Rust-knowing kernel developers would need to exist before this transition might begin; Jonathan Cameron answered: "a lot". I jumped in with some experience from the introduction of the ReStructured Text language for documentation; that process took longer than might have been expected, ran into some fierce (and ongoing) resistance, and is still not complete. It would not have succeeded to the degree it has without acceptance by a significant part of the development community. The introduction of Rust, which is a vastly more complex task, will not have an easier time of it; working to build wide acceptance is not optional. Pinchart said that most developers seem to appreciate the value of Rust now, but many of them worry about the cost of bringing it into the kernel. 

Ojeda said that the Rust for Linux developers need to either convince enough developers of the value of Rust so that Linus Torvalds will bless the change, or else simply end the discussion. He looked forward to the upcoming discussions at the Linux Plumbers Conference and Kernel Summit as a way to bring more developers into the process. I expressed some worries about relying too heavily on a blessing from Torvalds; that is a necessary condition but far from sufficient. 

Greg Kroah-Hartman said that he likes the idea of introducing Rust into kernel development, but said that this work has a long way to go yet. Device drivers (which is what the Rust developers are initially targeting) have to interact with many parts of the kernel, including the driver model, sysfs, and various other subsystems; the Rust to enable that interaction has not yet been written. There are going to be social, political, and technical issues around the introduction of Rust, and not even the technical issues have been taken care of yet. That said, he commended the Rust developers for the progress they have made so far. 

As the session ran out of time, Kroah-Hartman said that it would not be possible to require developers to write in Rust for at least five years. Pinchart asked whether anybody has thought about how many developers the community is willing to lose in this transition; the actual number will almost certainly not be zero. Ojeda concluded things by saying that the process for getting Rust into the kernel is the most important aspect, and that the next step for that will be the discussions at the Linux Plumbers Conference.
