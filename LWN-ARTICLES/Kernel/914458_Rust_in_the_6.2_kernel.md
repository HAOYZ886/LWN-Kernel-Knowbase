---
title: Rust in the 6.2 kernel
url: https://lwn.net/Articles/914458/
date: "November 17, 2022"
category: "Development tools-Rust"
author: "By Jonathan Corbet November 17, 2022"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 17, 2022

The merge window for the 6.1 release brought in [basic support](<https://lwn.net/Articles/910762/>) for writing kernel code in Rust — with an emphasis on "basic". It is possible to create a "hello world" module for 6.1, but not much can be done beyond that. There is, however, a lot more Rust code for the kernel out there; it's just waiting for its turn to be reviewed and merged into the mainline. Miguel Ojeda has now posted [the next round](<https://lwn.net/ml/linux-kernel/20221110164152.26136-1-ojeda@kernel.org/>) of Rust patches, adding to the support infrastructure in the kernel. 

This 28-part patch series is focused on low-level support code, still without much in the way of abstractions for dealing with the rest of the kernel. There will be no shiny new drivers built on this base alone. But it does show another step toward the creation of a workable environment for the development of code in the Linux kernel. 

As an example of how stripped-down the initial Rust support is, consider that the kernel has [eight different logging levels](<https://docs.kernel.org/next/core-api/printk-basics.html>), from "debug" through "emergency". There is a macro defined for each level to make printing simple; screaming about an imminent crash can be done with `pr_emerg()`, for example. The Rust code in 6.1 defines equivalent macros, but only two of them: [`pr_info!()` and `pr_emerg!()`](<https://elixir.bootlin.com/linux/v6.1-rc5/source/rust/kernel/print.rs#L151>); the macros for the other log levels were left out. The first order of business for 6.2 appears to be to fill in the rest of the set, from `pr_debug!()` at one end through `pr_alert!()` at the other. There is also `pr_cont!()` for messages that are pieced together from multiple calls. [This sample kernel module](<https://lwn.net/ml/linux-kernel/20221110164152.26136-5-ojeda@kernel.org/>) shows all of the print macros in action. 

A rather more complex macro added in this series is `#[vtable]`. The kernel makes extensive use of structures full of pointers to functions; these structures are at the core of the kernel's object model. A classic example is [`struct file_operations`](<https://elixir.bootlin.com/linux/latest/source/include/linux/fs.h#L2093>), which is used to provide implementations of the many things that can be done with an open file. The functions found therein vary from relatively obvious operations like `read()` and `write()` through to more esoteric functionality like `setlease()` or `remap_file_range()`. Anything in the kernel that can be represented as an open file provides one of these structures to implement the operations on that file. 

Operations structures like `file_operations` thus look a lot like [Rust traits](<https://doc.rust-lang.org/book/ch10-02-traits.html>), and they can indeed be modeled as traits in Rust code. But the kernel allows any given implementation to leave out any functions that are not relevant; a `remap_file_range()` operation will make no sense in most device drivers, for example. In the kernel's C code, missing operations are represented by a null pointer; code that calls those operations will detect the null pointer and execute a default action instead. Null pointers, though, are the sort of thing that the Rust world goes out of its way to avoid, so representing an operations structure in Rust requires some extra work. 

The `#[vtable]` macro is intended to perform the necessary impedance matching between C operations structures and Rust traits. Both the declaration of a trait and of any implementations will use this macro, so a trait definition will look like: 

```
#[vtable]
        pub trait Operations {
            /// Corresponds to the `open` function pointer in `struct file_operations`.
        	fn open(context: &Self::OpenData, file: &File) -> Result<Self::Data>;
        // ...
        }
```

While an implementation for a specific device looks like: 

```
#[vtable]
        impl kernel::file::Operations for some_driver {
            fn open(_data: &(), _file: &File) -> Result {
                Ok(())
            }
    	// ...
        }
```

If this implementation is to be passed into the rest of the kernel, it must be turned into the proper C structure. Rust can create the structure, but it is hard-put to detect which operations have been implemented and which should, instead, be represented by a null pointer. The `#[vtable]` macro helps by generating a special constant member for each defined function; in the above example, the `some_driver` type would have a constant `HAS_OPEN` member set to `true`. The code that generates the C operations structure can query those constants (at compile time) and insert null pointers for missing operations; the details of how that works can be seen in [this patch](<https://lwn.net/ml/linux-kernel/20221110164152.26136-7-ojeda@kernel.org/>). 

The submission for 6.2 adds `#[vtable]` but does not include any uses of it. The curious can see it in use by looking at [this large patch](<https://lwn.net/ml/linux-kernel/20220802015052.10452-18-ojeda@kernel.org/>) posted in August; searching for `#[vtable]` and `HAS_` will turn up the places where this infrastructure is used. 

Yet another new macro is `declare_err!()`, which can be used to declare the various error-code constants like `EPERM`. The 6.2 kernel will likely include [a full set of error codes](<https://lwn.net/ml/linux-kernel/20221110164152.26136-10-ojeda@kernel.org/>) declared with this macro rather than the minimal set included in 6.1. There is also [a mechanism](<https://lwn.net/ml/linux-kernel/20221110164152.26136-11-ojeda@kernel.org/>) to translate many internal Rust error into Linux error codes. 

The Rust [`Vec` type](<https://doc.rust-lang.org/std/vec/struct.Vec.html>) implements an array that will grow as needed to hold whatever is put into it. Growing, of course, involves memory allocation, which can fail in the kernel. In 6.2, `Vec` as implemented in the kernel will likely have two methods called `try_with_capacity()` and `try_with_capacity_in()`. They act like the standard [`with_capacity()`](<https://doc.rust-lang.org/std/vec/struct.Vec.html#method.with_capacity>) and [`with_capacity_in()`](<https://doc.rust-lang.org/std/vec/struct.Vec.html#method.with_capacity_in>) `Vec` methods in that they preallocate memory for data to be stored later, but with the difference that they can return a failure code. The `try_` variants will allow kernel code to attempt to allocate `Vec`s of the needed size and do the right thing if the allocation fails, rather than just calling `panic()` like the standard versions do. 

One of the more confusing aspects of Rust for a neophyte like your editor is the existence of two string types: [`str`](<https://doc.rust-lang.org/std/primitive.str.html>) and [`String`](<https://doc.rust-lang.org/std/string/struct.String.html>); the former represents a borrowed reference to a string stored elsewhere, while the latter actually owns the string. The kernel's Rust support will define two variants of those, called [`CStr`](<https://lwn.net/ml/linux-kernel/20221110164152.26136-17-ojeda@kernel.org/>) and [`CString`](<https://lwn.net/ml/linux-kernel/20221110164152.26136-22-ojeda@kernel.org/>), which serve the same function for C strings. Specifically, they deal with a string that is represented as an array of bytes and terminated with a `NUL` character. Rust code that passes strings into the rest of the kernel will need to use these types. 

The series ends with a grab-bag of components that will be useful for future additions. The [`dbg!()` macro](<https://lwn.net/ml/linux-kernel/20221110164152.26136-24-ojeda@kernel.org/>) makes certain types of debugging easier. There is [code for compile-time assertions](<https://lwn.net/ml/linux-kernel/20221110164152.26136-25-ojeda@kernel.org/>) and to [force build errors](<https://lwn.net/ml/linux-kernel/20221110164152.26136-26-ojeda@kernel.org/>). The [`Either` type](<https://lwn.net/ml/linux-kernel/20221110164152.26136-28-ojeda@kernel.org/>) can hold an object that can be either one of two distinct types. Finally, the [`Opaque` type](<https://lwn.net/ml/linux-kernel/20221110164152.26136-29-ojeda@kernel.org/>) is for structures used by the kernel that are never accessed by Rust code. Using this type can improve performance by removing the need to zero-initialize the memory holding it before calling the initialization function. 

As can be seen, these patches are slowly building the in-kernel Rust code up so that real functionality can be implemented in Rust, but this process has some ground to cover yet. It's not clear whether more Rust code will be proposed for 6.2, or whether this is the full set. The pace of change may seem slow to developers who would like to start doing real work in Rust, but it does have the advantage of moving in steps that can be understood — and reviewed — by the kernel community. The Rust-for-Linux work has been underway for a few years already; getting up to full functionality may well take a while longer yet.
