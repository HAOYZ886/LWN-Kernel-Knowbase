---
title: Upcoming Rust language features for kernel development
url: https://lwn.net/Articles/1039073/
date: "October 8, 2025"
category: "Development tools-Rust"
author: "By Daroc Alden October 8, 2025 Kangrejos"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Daroc Alden**  
October 8, 2025

* * *

[Kangrejos](<https://lwn.net/Articles/1039334/>)

The [ Rust for Linux](<https://rust-for-linux.com/>) project has been good for Rust, Tyler Mandry, one of the co-leads of Rust's language-design team, said. He gave a talk at [ Kangrejos 2025](<https://kangrejos.com/>) covering upcoming Rust language features and thanking the Rust for Linux developers for helping drive them forward. Afterward, Benno Lossin and Xiangfei Ding went into more detail about their work on the three most important language features for kernel development: field projections, in-place initialization, and arbitrary self types. 

Many people have remarked that the development of new language features in Rust can be quite slow, Mandry said. Partly, that can be attributed to the care the Rust language team takes to avoid enshrining bad designs. But the biggest reason is ""alignment in attention"". The Rust project is driven by volunteers, which means that if there are not people focusing on pushing a given feature or group of related features forward, they languish. The Rust for Linux project has actually been really helpful for addressing that, Mandry explained, because it is something that a lot of people are excited about, and that focuses effort onto the few specific things that the Linux kernel needs. 

[ ![\[Tyler Mandry\]](https://static.lwn.net/images/2025/tyler-mandry-kangrejos-small.png) ](<https://lwn.net/Articles/1040419>)

Mandry then went through a whirlwind list of upcoming language features, including [ types without known size information](<https://github.com/Skepfyr/rfcs/blob/extern-types-v2/text/3396-extern-types-v2.md>), [ reference-counting improvements](<https://github.com/joshtriplett/rfcs/blob/use/text/3680-use.md>), [ user-defined function modifiers of the same kind as `const`](<https://internals.rust-lang.org/t/pre-pre-rfc-generic-effect-system-for-functions/22550?utm_source=chatgpt.com>), and more. At the end, he asked which of those were most important to Rust for Linux, and how the assembled kernel developers would prioritize them. Beyond the three features to be discussed later, Lossin said that the project definitely wanted the ability to write functions that can be evaluated at compile time (called [ `const` functions](<https://doc.rust-lang.org/std/keyword.const.html#compile-time-evaluable-functions>) in Rust) in trait definitions. Danilo Krummrich asked for [ specialization](<https://github.com/rust-lang/rust/issues/31844>), which immediately prompted an ""Oh no!"" from Lossin, due to the feature's nearly decade-long history of causing problems for Rust's type system. Specialization would allow two overlapping implementations for a single trait to exist, with the compiler picking the more specific one. Matthew Maurer asked for some ability to control what the compiler does on integer overflow. 

Ultimately, Miguel Ojeda told Mandry that the priority should be on stabilizing the unstable language features that Rust for Linux currently uses, followed by language features that would change how the project structures its code, followed by everything else. The next two talks went into much more detail about the current status and future plans for some of those key language features. 

#### Field projections

Field projection refers to the idea of taking a pointer to a structure, and turning it into a pointer to a field of the structure. Rust does already have this for the built-in reference and pointer types, but it can't always be made to work for user-defined smart-pointer types. Since the Rust for Linux developers would like to have custom smart pointers to handle [ untrusted data](<https://lwn.net/Articles/1034603/>), reference counting, external locking, and related kernel complications, they would benefit from a general language feature allowing field projections for all pointer types using the same syntax. Lossin spoke about his work on the problem, which has been ongoing since Kangrejos 2022\. There has been ""lots of progress"" so far, but the work is still in the design stage, with a few details left to work out. 

The built-in field projections all have the same kind of type signature, Lossin explained. For example, the code for converting a reference to an object into a reference to one of its fields and the code for converting a raw pointer to an object into a raw pointer to one of its fields look different, but have similar signatures: 

```
fn project_reference(r: &MyStruct) -> &Field {
            &r.field
        }
    
        unsafe fn project_pointer(r: *mut MyStruct) -> *mut Field {
            unsafe { &raw mut (*r).field }
        }
    
        // The equivalent C code would look like this:
        struct field *project(struct my *r) {
            return &(r->field);
        }
```

This example uses the relatively recent [ raw borrow](<https://github.com/rust-lang/rfcs/blob/master/text/2582-raw-reference-mir-operator.md>) syntax. 

The [ `Pin`](<https://rust.docs.kernel.org/core/pin/struct.Pin.html>) type throws a bit of a wrench into things. The Rust compiler is, by default, free to move structures around for performance reasons. That doesn't work when the structure is being referenced from the C side, so the `Pin` type is used to mark structures that shouldn't be moved. Projecting a ~~`Pin <MyStruct>`~~ `Pin<&mut MyStruct>` [Lossin sent LWN a correction: `Pin` is always used to wrap a pointer type, not a structure directly] might produce either a `Pin<&mut Field>` or a plain `&mut Field` depending on whether the field is also of a type that shouldn't be moved or not. So the most general possible signature for the field projection operation would be something like this, Lossin said: 

```
Container<'a, Struct> -> Output<'a, Field>
```

That is, given some pointer type that wraps a structure and must be valid for lifetime `a`, projecting a field gives a (possibly different) output pointer type wrapping a field of that structure, valid for the same lifetime. Lossin then gave an example of how supporting this could make fully implementing read-copy-update (RCU) support in the kernel's Rust bindings a lot easier. 

[ ![\[Benno Lossin\]](https://static.lwn.net/images/2025/benno-lossin-kangrejos-small.png) ](<https://lwn.net/Articles/1040419#benno>)

The RCU mechanism protects readers from concurrent writers, he explained, but it doesn't protect writers from each other. It's somewhat common in the kernel, therefore, to have a mutex protecting some data, with a frequently accessed field of that data being protected by RCU. That way, readers rely on the RCU lock (which is cheap), and writers synchronize with each other using the mutex. Translating that interface to Rust poses problems: Rust doesn't allow any access to the content inside a [ `Mutex`](<https://rust.docs.kernel.org/kernel/sync/lock/mutex/type.Mutex.html>) without locking it first, so the straightforward translation of this pattern wouldn't work. It would force Rust readers to lock the mutex in order to read the RCU field, which would be an unacceptable performance hit. 

With generalized field projection in the language, though, the Rust for Linux developers could write bindings that permit projecting a `&Mutex<MyStruct>` into an `&Rcu<Field>` without holding the lock. In driver code, attempting to read from the RCU-protected field would look like a normal access, the same way it is in C — but the compiler would still check that the other, non-RCU-protected data isn't touched without holding the mutex. 

Lossin ended by asking the assembled developers to keep an eye on [the tracking issue](<https://github.com/rust-lang/rust/pull/146307>) for the feature, and provide feedback on it. Daniel Almeida asked whether testing the feature outside the mainline kernel was really helpful; Ojeda affirmed that it was, because that makes it easier to go to the Rust team and make a case to stabilize the feature. The Rust for Linux project is trying not to use any new unstable features (and to compile with a version of Rust equal to or older than the version packaged on Debian stable), so the feature needs to be completed and make it into Debian 14 (expected in 2027) before it will be widely usable in kernel code. 

Andreas Hindborg asked: ""Can we have this yesterday, please?"", to general amusement. The kernel's Rust bindings already feature a plethora of custom pointers encoding various invariants; this feature, whenever it becomes available to kernel code, may make them a good deal easier to use in driver code. 

#### Arbitrary self types

Ding gave an update immediately afterward about another ergonomic language feature for custom pointers: arbitrary self types. In Rust, a method on a type can have a first argument that is an object of the type or that is a reference to one. Such a method can be called with the `.method()` syntax, instead of the more general `Type::function()` syntax. But the proliferation of smart pointers in kernel Rust code means that the programmer frequently does not have a plain reference; often, they instead have a `Pin`, an [ `Arc`](<https://rust.docs.kernel.org/kernel/sync/struct.Arc.html>), or some other smart-pointer type. 

The arbitrary self types proposal that Ding has been working on would let programmers write methods that take smart pointers, instead of normal references: 

```
impl MyStruct {
            fn method(self: Pin<&mut MyStruct>) {}
        }
```

Unfortunately, adding this to the compiler has not proved to be straightforward. The interaction with Rust's existing [ `Deref` trait](<https://rust.docs.kernel.org/core/ops/trait.Deref.html>), which makes custom smart pointers possible in the first place, complicates the implementation because not all of the type information is available while searching for matching methods. Currently, if the user has a `Pin<&mut MyStruct>` and they call a method on it, the compiler will first look for a matching method for `Pin`. If one isn't found, it will try to dereference the type, producing a `&mut MyStruct`. That type is checked for matching methods, and then is dereferenced one final time, producing a `MyStruct`. That type will finally have a matching method, or else the compiler will emit a type error. 

[ ![\[Xiangfei Ding\]](https://static.lwn.net/images/2025/xiangfei-ding-kangrejos-small.png) ](<https://lwn.net/Articles/1040419#xiangfei>)

By the time that procedure begins checking functions associated with `MyStruct`, it will have already discarded information about the wrapping types, which an implementation of arbitrary self types needs. Ding spent a few minutes explaining the approaches for rectifying the problem that he had tried and discarded, before focusing on the current approach. He has added another trait — tentatively called `Receiver` — that is used to mark types that can be used with arbitrary self types. Then the compiler can try following the chain of `Receiver` implementations before following the chain of `Deref` implementations. That does mean that a pointer type will have to opt into being used as an arbitrary self type, but Ding didn't see that as a downside. Letting the author of a pointer type decide when it should support the new feature eliminates a lot of concerns around accidentally introducing backward compatibility problems. For the kernel, it doesn't really impose a barrier, because the Rust developers can just add `Receiver` implementations as they run across cases that require them. 

Ojeda asked how long Ding thought it would take to finalize the arbitrary self types feature; in particular, would it be ready within a year? Ding agreed that a year was possible, although he would need support from the Rust language team in order to make that happen. He wants to run [ Crater](<https://github.com/rust-lang/crater?tab=readme-ov-file#crater>), the tool that the Rust community uses to check whether compiler changes break any published Rust libraries, against his change before submitting the code. Ojeda offered help with obtaining a large build machine to do that, since Ding has had trouble previously with the memory requirements to compile some packages during a Crater run. 

#### In-place initialization

The other topic that Ding wanted to cover was his work on in-place initialization. Like the other new language features being discussed, this doesn't really enable new use cases, but it does make common kernel code cleaner. Currently, Rust code in the kernel uses the [ `pin_init!()` macro](<https://rust.docs.kernel.org/kernel/prelude/macro.pin_init.html>) to create structures that are fixed in place after initialization (by being wrapped in `Pin`). 

There's nothing wrong with `pin_init!()`: ""We love `pin_init!()`! We want to make a language feature out of it."" Adopting a language feature for in-place initialization would also help with a handful of sharp edges outside kernel code; it could make creating large [ `Future`](<https://rust.docs.kernel.org/core/future/trait.Future.html>) values on the heap more ergonomic, and let some traits become [ dyn-compatible](<https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility>). The exact design of this language feature was more up in the air; Ding covered three different proposals for how it could work. 

The simplest, proposed by Alice Ryhl and Lossin, would be to add a new keyword, `init`, before a structure-initializing expression in order to ask the compiler to automatically write an implementation of the kernel's [ `PinInit` trait](<https://rust.docs.kernel.org/kernel/prelude/trait.PinInit.html>). That has the nice benefit of being a fairly minimal change to the language, although it would lock in the use of the `PinInit` trait in its current form. 

Another solution, [ proposed](<https://hackmd.io/@rust-lang-team/r1zNmpwwgl#In-place-initialization-via-outptrs>) by Taylor Cramer, would introduce a new type of reference into the language. Rust's existing references can either be read from (`&`) or read from and written to (`&mut`). This proposal would add a third type, `&out`, that can only be written to, not read from. The only way to use an `&out` reference would be to either write to it, or use projection to break it apart into multiple `&out` references to various fields. Under this scheme, in-place initialization would look like allocating space on the heap, and then returning an `&out` reference. The calling code could then fill it in however it wants to, potentially passing off sub-parts to other functions. The compiler would track that the `&out` references are all used before allowing the code to obtain a normal `&mut` reference to the heap allocation. 

That proposal was considerably less polished than Ryhl and Lossin's approach, however. Ding later told me that he, Mandry, and other compiler contributors at Kangrejos were actually working on figuring out how it would interact with some of the Rust compiler's internals in between talks that day. By the end of the conference, they had a rough idea of how it could be implemented, so a more detailed version of the out-pointer proposal may be forthcoming shortly. 

The final design, taking inspiration from C++, would be a form of guaranteed optimization, where constructing a new value and then immediately moving it to the heap causes it to be constructed on the heap in the first place. Ding was less sure about the details of the final proposal; he suggested that the best way forward might be to implement both the `PinInit` proposal and the out-reference proposal, and see how well each approach works in practice. 

Regardless of which approach ends up being chosen, it seems clear that Mandry's point about the Rust for Linux project driving language improvement is correct. While these features are in the early stages, adopting them could significantly simplify code involving user-defined smart pointers, both within and outside the kernel. 

**Update:** Since the talks described in this article, the work on field projection has received an update. Lossin wrote in to inform LWN that all fields of all structures are now considered structurally pinned, so projecting a `Pin` will now always produce a `Pin<&mut Field>` or similar value.
