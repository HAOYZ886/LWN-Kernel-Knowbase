---
title: A first look at Rust in the 6.1 kernel
url: https://lwn.net/Articles/910762/
date: "October 13, 2022"
category: "Development tools-Rust; Releases-6.1"
author: "By Jonathan Corbet October 13, 2022"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 13, 2022

There have been a lot of significant changes merged into the mainline for the 6.1 release, but one of the changes that has received the most attention will also have the least short-term effect for users of the kernel: the introduction of support for the Rust programming language. No system with a production 6.1 kernel will be running any Rust code, but this change does give kernel developers a chance to play with the language in the kernel context and get a sense for how Rust development feels. Perhaps the most likely conclusion for most developers, though, will be that there isn't yet enough Rust in the kernel to do much of anything interesting. 

Work on Rust for the Linux kernel has been going on for a few years, and it has resulted in the creation of a lot of support code and [some interesting drivers](<https://lwn.net/Articles/907685/>) to look at. There are other initiatives underway, including the writing of [an Apple graphics driver](<https://lwn.net/ml/rust-for-linux/70657af9-90bb-ee9e-4877-df4b14c134a5@asahilina.net/>) in the Rust language. For the initial merge into the mainline kernel, though, Linus Torvalds [made it clear](<https://lwn.net/Articles/908347/>) that as little functionality as possible should be included. So those drivers and their support code were trimmed out and must wait for a future kernel release. What _is_ there is the support needed to build a module that can be loaded into the kernel, along with a small sample module. 

#### Building Rust support

The first challenge that interested developers will run into is actually building that support. The kernel configuration process looks for the prerequisites on the build system and, if they are not present, silently disables the Rust options so that they will not even show in, for example, `make menuconfig`. Your editor, despite having Rust installed on the system in question, ran into this and was thus forced into the ignominious process of actually reading [the documentation](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/rust/quick-start.rst>) to figure out what was missing. 

Building the Rust support requires specific versions of the Rust compiler and bindgen utility — specifically, Rust 1.62.0 and bindgen 0.56.0. If the target system has newer versions, the configuration process will emit warnings but will proceed anyway. More awkwardly for anybody who is trying to do the build with the Rust toolchain provided by their distributor, the build process also needs the Rust standard library source so that it can build its own version of the `core` and `alloc` crates. Until distributors start shipping "Rust for the kernel" packages, getting that code into a place where the build process will find it will be a bit awkward. 

The way to easily obtain that dependency is to throw in the towel, drop the distributor's toolchain, and install everything from the Rust repositories instead. The ["getting started" page](<https://www.rust-lang.org/learn/get-started>) describes how to do this; inevitably, it involves one of those confidence-building "`curl|bash`" operations. The installer is entirely uninterested in _where_ one might like one's Rust stuff installed (it goes into `~/.cargo`) and silently modifies the user's Bash startup scripts to add the new directory into the `PATH` variable. The end result does work, though, and makes it easy to install the needed dependencies. 

#### The sample module

Once that is done, the kernel configuration system will consent to set the `CONFIG_RUST` option; an additional option will build the sample module. That module ([`samples/rust/rust_minimal.rs`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/samples/rust/rust_minimal.rs>)) is minimal indeed, but it is enough to get a sense for what kernel code in Rust will look like. It starts with the Rust equivalent of a `#include` line: 

```
use kernel::prelude::*;
```

The pulls in the declarations found in [`rust/kernel/prelude.rs`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/rust/kernel/prelude.rs>), making a few types, functions, and macros available. 

A kernel module written in C includes a number of calls to macros like `MODULE_DESCRIPTION()` and `MODULE_LICENSE()` that stash metadata about the module in a separate ELF section. The `module_init()` and `module_exit()` macros identify the module's constructor and destructor functions, respectively. The Rust equivalent puts much of that boilerplate into a single macro call: 

```
module! {
            type: RustMinimal,
            name: b"rust_minimal",
            author: b"Rust for Linux Contributors",
            description: b"Rust minimal sample",
            license: b"GPL",
        }
```

This macro is fussy about the ordering of the various fields and will complain if the developer gets that wrong. Beyond putting all of this information into a single call, the `module!` macro includes a `type:` entry which will be the pointer to the actual module code. The developer will be expected to supply a type that does something interesting. In the sample module, that type looks like this: 

```
struct RustMinimal {
            numbers: Vec<i32>,
        }
```

It is a simple structure containing a `Vec` (an array, more or less) of 32-bit integer values. That's pretty boring on its own, but Rust then allows the addition of interface ("trait") implementations to a structure type. So the sample module implements the `kernel::Module` trait for the `RustMinimal` type: 

```
impl kernel::Module for RustMinimal {
            fn init(_module: &'static ThisModule) -> Result<Self> {
                pr_info!("Rust minimal sample (init)\n");
                pr_info!("Am I built-in? {}\n", !cfg!(MODULE));
        
                let mut numbers = Vec::new();
                numbers.try_push(72)?;
                numbers.try_push(108)?;
                numbers.try_push(200)?;
    
                Ok(RustMinimal { numbers })
            }
        }
```

The `init()` function is expected to do the usual module initialization work. In this case, it babbles a bit to the system log (showing off in the process the `cfg!()` macro that can be used to query kernel-configuration parameters at compile time). It then allocates a mutable `Vec` and attempts to put three numbers into it. The use of `try_push()` is important here: a `Vec` will resize itself when necessary. That involves allocating memory, which can fail in the kernel environment. Should that allocation fail, `try_push()` will return a failure status and that, in turn, will cause `init()` to return failure (that is what the "`?`" at the end of the line does). 

Finally, if all goes well, it returns a `RustMinimal` structure with the allocated `Vec` and a success status. Since this module has not interacted with any other kernel subsystems, it won't actually do anything other than wait patiently to be removed. There isn't a function for module removal in the `Kernel::Module` trait; instead, a simple destructor for the `RustMinimal` type is used: 

```
impl Drop for RustMinimal {
            fn drop(&mut self) {
                pr_info!("My numbers are {:?}\n", self.numbers);
                pr_info!("Rust minimal sample (exit)\n");
            }
        }
```

This function prints out the numbers that were stored in the `Vec` at initialization time (thus confirming that the data survived in meantime) and returns; after that, the module will be removed and its memory freed. There does not appear to be a way for module removal to fail — which occasionally needs to happen in real-world modules. 

#### Beyond "hello world"

That is, to a first approximation, the extent of what can be done with Rust kernel modules in 6.1. Torvalds asked for something that could do "hello world" and that is what we got. It is something that can be played with, but it cannot be used for any sort of real kernel programming at this point. 

That situation will, hopefully, change in the near future. The next step for the Rust-for-Linux developers will be to start adding some of the infrastructure they have created to interface with other kernel subsystems. That will allow for the writing of some real kernel code and, just as importantly, show what the abstractions needed to work with other kernel subsystems will look like. This needs to happen soon; Rust in the kernel has some momentum now, but that could be lost if it remains limited to printing kernel log messages for any period of time. With luck, Rust in the 6.2 kernel will be significantly more capable.
