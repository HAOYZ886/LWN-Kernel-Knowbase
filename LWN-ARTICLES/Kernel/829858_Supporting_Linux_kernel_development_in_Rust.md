---
title: Supporting Linux kernel development in Rust
url: https://lwn.net/Articles/829858/
date: "August 31, 2020"
category: "Development tools-Rust"
author: "August 31, 2020 This article was contributed by Nelson Elhage LPC"
---

August 31, 2020

This article was contributed by Nelson Elhage

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2020-Linux_Plumbers_Conference>)

The [Rust programming language](<https://www.rust-lang.org/>) has long aimed to be a suitable replacement for C in operating-system kernel development. As Rust has matured, many developers have expressed growing interest in using it in the Linux kernel. At the 2020 (virtual) [Linux Plumbers Conference](<https://linuxplumbersconf.org/>), the [LLVM](<https://llvm.org/>) microconference track hosted a session on open questions about and obstacles to accepting Rust upstream in the Linux kernel. The interest in this topic can be seen in the fact that this was the single most heavily attended session at the 2020 event. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

This session built on prior work by many developers, including a [talk last year by Alex Gaynor and Geoffrey Thomas [YouTube]](<https://www.youtube.com/watch?v=RyY01fRyGhM>) at the Linux Security Summit. At that talk, they presented their work prototyping Rust kernel modules and made the case for adopting Rust in the kernel. They focused on security concerns, citing work showing that around two-thirds of the kernel vulnerabilities that were assigned CVEs in both Android and Ubuntu stem from memory-safety issues. Rust, in principle, can completely avoid this error class via safer APIs enabled by its type system and [borrow checker](<https://doc.rust-lang.org/1.8.0/book/references-and-borrowing.html>). 

Since then, Linus Torvalds and other core kernel maintainers have expressed [openness in principle](<https://www.theregister.com/2020/06/30/hard_to_find_linux_maintainers_says_torvalds/>) to supporting kernel development in Rust, so the session at Plumbers aimed to work through some of the requirements to eventually allowing Rust in-tree. The session was proposed and discussed [on the linux-kernel mailing list](<https://lwn.net/ml/linux-kernel/CAKwvOdmuYc8rW_H4aQG4DsJzho=F+djd68fp7mzmBp3-wY--Uw@mail.gmail.com/>), where some of the topics of discussion were previewed. 

This session, too, featured Thomas and Gaynor, along with Josh Triplett — the Rust language team co-leader and a longtime Linux kernel developer — and a number of other interested developers. They briefly touched on their work so far and some of their initial thoughts and questions before opening the bulk of the time to discussion. They gave a brief example of what kernel-mode Rust code might look like (from Thomas and Gaynor's [linux-kernel-module-rust](<https://github.com/fishinabarrel/linux-kernel-module-rust/>) project). 

The speakers emphasized that they are not proposing a rewrite of the Linux kernel into Rust; they are focused only on moving toward a world where new code may be written in Rust. The ensuing conversation focused on three areas of potential concern for Rust support: making use of the existing APIs in the kernel, architecture support, and a question about ABI compatibility between Rust and C. 

#### Binding to existing C APIs

In order to be useful for kernel development, it's not enough that Rust is able to generate code that can be linked into the kernel; there also needs to be a way for Rust to access the vast number of APIs used in the Linux kernel, which are all presently defined in C header files. Rust has good support for interoperating with C code, including support for both calling functions using the C ABI and for defining functions with C-compatible ABIs that can be called from C. Furthermore, the [bindgen](<https://github.com/rust-lang/rust-bindgen>) tool is capable of parsing C header files to produce the appropriate Rust declarations, so that Rust does not need to duplicate definitions from C, which also provides a measure of cross-language type checking. 

> [![\[Rust discussion\]](https://static.lwn.net/images/conf/2020/lpc/rust1-strip.png)](<https://lwn.net/Articles/829942/>)

On the surface, these features make Rust well-equipped to integrate with existing C APIs, but the devil is in the details, and both the work to date and the conversation at the session revealed a handful of open challenges. For example, Linux makes heavy use of preprocessor macros and inline functions, which aren't easily supported by bindgen and Rust's foreign-function interface. 

The ubiquitous `kmalloc()` function, for instance, is defined as `__always_inline`, meaning that it is inlined into all of its callers and no `kmalloc()` symbol exists in the kernel symbol table for Rust to link against. This problem can be easily worked around — one can define a `kmalloc_for_rust()` symbol containing an un-inlined version — but performing these workarounds by hand would result in a large amount of manual work and duplicated code. This work could potentially be automated by an improved version of `bindgen`, but such a tool does not yet exist. 

The conversation also touched on a second question about API bindings: how much will C APIs need to be manually "wrapped" to present idiomatic Rust interfaces? A look at two existing Rust kernel module projects gives a flavor for some of the choices here. 

In the linux-kernel-module-rust project, pointers into user space are wrapped into a `UserSlicePtr` type, which ensures appropriate use of `copy_to_user()` or `copy_from_user()`. This wrapper provides a level of safety in Rust code (these pointers can't be dereferenced directly), and also makes Rust code more idiomatic; writing to a user-space pointer looks something like 

```
user_buf.write(&kernel_buffer)?;
```

The `?` here is part of Rust's [error-handling machinery](<https://doc.rust-lang.org/edition-guide/rust-2018/error-handling-and-panics/the-question-mark-operator-for-easier-error-handling.html>); this style of returning and handling errors is ubiquitous in Rust. Such wrappers make the resulting Rust more familiar to existing Rust developers, and enable Rust's type system and borrow checker to provide a maximum amount of safety. However, they must be carefully designed and developed for each API, which is a lot of work and creates distinct APIs for modules written in C and Rust. 

[John Baublitz's demo module](<https://github.com/jbaublitz/knock-out>), instead, binds the kernel's user-access functions more directly; the corresponding code there looks something like: 

```
if kernel::copy_to_user(buf, &kernel_buffer[0..count]) != 0 {
       	return -kernel::EFAULT;
        }
```

This style is easy to implement — the bindings are largely autogenerated by bindgen — and would also be more comfortable for existing kernel developers who have to review or patch Rust code. However, the code is much less idiomatic for Rust developers, and potentially gives up a lot of the safety guarantees that Rust promises. 

There was some agreement at the session that writing Rust wrappers will make sense for some of the most common and critical APIs, but that manually wrapping every kernel API would be infeasible and undesirable. Thomas mentioned that Google is working on automatically generating idiomatic bindings to C++ code, and pondered whether the kernel could do something similar, perhaps building on top of existing [sparse](<https://www.kernel.org/doc/html/v4.12/dev-tools/sparse.html>) annotations or some new annotations added to the existing C to guide the binding generator. 

#### Architecture support

The next area of discussion was architecture support. At present, the only mature Rust implementation is the [rustc](<https://doc.rust-lang.org/rustc/what-is-rustc.html>) compiler, which emits code via LLVM. The Linux kernel supports a wide range of architectures, several of which have no available LLVM backend. For a few others, an LLVM backend exists, but rustc does not yet support that backend. The presenters wanted to understand whether full architecture support was a blocker to enabling Rust in the kernel. 

Several people said that it would be acceptable to implement drivers in Rust that would never be used on the more obscure architectures anyway. Triplett suggested that adding Rust into the kernel would help drive increased architecture support for Rust, citing his experience with the Debian project. He mentioned that introducing Rust software into Debian helped to motivate enthusiasts and users of niche architectures to improve Rust support, and he expected that adding support to the kernel would have a similar effect. In particular, he was confident that any architecture with an LLVM backend would quickly be supported in rustc. 

The conversation also discussed alternate Rust implementations as a path toward broader architecture support. The [mrustc](<https://github.com/thepowersgang/mrustc>) project is an experimental Rust compiler that emits C code. Using mrustc would potentially let Rust be compiled via the same C compiler that was compiling the rest of the kernel. 

In addition, Triplett cited some interest in — and work toward — a Rust front end for GCC, potentially enabling Rust to target any architecture GCC supports. This project is in an early stage, but it presents another avenue toward closing the architecture gap in the future. The conclusion from this section was a little uncertain, but there did not seem to be strong pushback against the idea of supporting Rust device drivers without waiting for broader architecture support. 

#### ABI compatibility with the kernel

Gaynor also asked for advice on a question of ABI compatibility. Since Rust is (currently) compiled via LLVM, and the kernel is most commonly built with GCC, linking Rust code into the kernel may mean mixing code emitted by GCC and LLVM. Even though LLVM aims to be ABI-compatible with GCC, there has been some pushback based on concerns that this strategy created a risk of subtle ABI incompatibilities. The presenters wondered whether the kernel community would prefer to limit Rust support to kernels built with Clang in order to ensure compatibility. 

Greg Kroah-Hartman confirmed that the current kernel rule was that compatibility is only guaranteed if all object files in the kernel are built with the same compiler, using identical flags. However, he also expressed comfort with linking LLVM-built Rust objects into a GCC-built kernel as long as the objects are built at the same time, with the appropriate options set, and the resulting configurations are fully tested. He did not feel the need for any additional restrictions until and unless actual problems arise. Florian Weimer clarified that ABI issues tend to be in obscure corners of the language — for instance, returning a struct containing a bitfield by value — and that he would expect that the core, commonly-used parts of the ABI should pose no compatibility problems. 

Triplett emphasized that calling between GCC and Rust was routine and widespread in user space, and so from the Rust side he has no concerns about compatibility. It sounded like this concern should not, in the end, be an impediment to bringing Rust into the kernel. 

#### Conclusions

The session ended without any further specific next steps, but it seems that, overall, there is enthusiasm for eventually supporting Rust modules along with increasing agreement on the broad requirements for that support. The next big step will likely be when someone proposes a real Rust driver for inclusion into the kernel. A concrete use case and implementation always helps to force clarity about any remaining contentious questions and design decisions.
