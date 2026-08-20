---
title: "Kernel hackers at Cauldron, 2025 edition"
url: https://lwn.net/Articles/1039784/
date: "October 2, 2025"
category: "Development tools-Compiler toolchain; Tracing"
author: "By Jonathan Corbet October 2, 2025 Cauldron"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 2, 2025

* * *

[Cauldron](<https://lwn.net/Archives/ConferenceIndex/#GNU_Tools_Cauldron-2025>)

The GNU Tools Cauldron is almost entirely focused on user-space tools, but kernel developers need a solid toolchain too. In what appears to be a developing tradition ([started in 2024](<https://lwn.net/Articles/990379/>)), some kernel developers attended the [2025 Cauldron](<https://conf.gnu-tools-cauldron.org/opo25/>) for the second year in a row to discuss their needs with the assembled toolchain developers. Topics covered in this year's gathering include Rust, better [BPF type format (BTF)](<https://www.kernel.org/doc/html/latest/bpf/btf.html>) support, SFrame, and more. 

#### Rust

The initial topic, guided by Miguel Ojeda over a remote link, was support for Rust code in the kernel. The Debian project, it seems, is the first to run into a problem that is likely to be felt more widely: the difficulties facing anyone who needs to compile out-of-tree modules written in Rust. As a general rule, loadable kernel modules should be built with the same version of the compiler that was used for the main kernel, but efforts (including the ["modversions" mechanism](<https://lwn.net/Articles/986892/>)) have long been made to loosen that requirement. So it will usually work to build a module with whichever version of the compiler happens to be handy. 

The Rust compiler is different, though. When compiling code, it creates a number of files that serve in a role similar to that of C header files, and it will consume those files when building other translation units. But the Rust compiler insists on an exact version match or it will be unwilling to use those files, causing compilation to fail. That makes the building of out-of-tree modules nearly impossible if they involve Rust code. 

The general consensus in the room was that this was a problem that needed to be fixed in the compiler. Since, for now, the only viable compiler for kernel code is `rustc`, there was not much that the assembled GCC developers could do about it. José Marchesi asked whether `gccrs`, the under-development GCC front-end for Rust, would have the same problem; how flexible that compiler will be on this point is unclear, and nobody had an answer to that question. 

#### Inline support for BTF

Alan Maguire, also participating remotely, raised the topic of support for inline call sites in BTF, which describes the types of functions and data structures in the kernel. When the kernel is built, inline functions are directly substituted into the code by the compiler, so there is no separate call site in the resulting binary; that can make tracing those calls more difficult. Representing these call sites, of which there are over 400,000 in a built kernel, would make debugging and performance analysis easier. 

Getting there, he said, requires adding three new types to BTF. The `BTF_KIND_LOC_PARAM` type indicates where a parameter to an inline function call can be found; objects of this type are gathered together under the `BTF_KIND_PROTO` type, which contains all of the parameters to a call. That, in turn, is pulled into an object of the `BTF_KIND_LOCSEC` type with the function names, prototypes, and address information. Generating this information in the build process is not a sure thing; Maguire was able to collect full information for 65% of the inline function calls, and partial information (only some of the parameters) for another 17%. After deduplication, the result is 9.5MB of collected BTF data. 

Maguire is planning on making this data available via a new virtual file, such as `/sys/kernel/btf/vmlinux.extra`, to avoid bloating the ordinary BTF data with the inline information. There is also a way to build this information into a separate loadable module so that it need not be resident in memory when not in use. 

There was seemingly more to be said about this topic but, at this point, the remote link, which was hosted on a free-of-charge but proprietary service, hit its time limit and abruptly shut down, so the conversation moved on. 

#### Tracing and related

Alexei Starovoitov said that he would like to have a way to inject some assembly code at the call sites of inline functions, providing a hook that would be easy to attach to for tracing purposes. This would make the arguments to the function available, even if they have been optimized out by the compiler. Jakub Jelinek said that inserting the code is easy, but that the compiler's optimization passes can split up an inline function's code and spread it around, making the notion of a specific call site a bit fuzzy. Steve Rostedt said that would make exit tracing especially difficult to implement. Paul McKenney added that this kind of optimization could end up reordering calls to multiple inline functions, adding more potential for confusion. 

Segher Boessenkool pointed out that it will often be difficult to reconstruct the arguments to an inline function; part of the whole point of inlining, he said, was to be able to perform global optimizations. 

Marchesi asked for an update on which version of [SFrame](<https://lwn.net/Articles/1029189/>) can be expected to land in the kernel. Indu Bhagat said that the deferred unwinding support needed to fully implement SFrame is in the kernel now, but the other pieces have not yet been merged. When the SFrame-specific code lands, it will support the upcoming version-3 specification. That version is also supported by binutils 2.46, which is expected in January. Rostedt said that the SFrame code is waiting for toolchain support, so it will not land in 6.18; it is likely to show up in 6.19 or 6.20. Bhagat added that LLVM will have SFrame support sometime in the (northern-hemisphere) spring. 

Nick Clifton, the binutils maintainer, said that he would be willing to move the next binutils release forward to December if that would help, but Rostedt said that there was not that much urgency. Finishing the new version of the SFrame specification needs to happen first, as does the design of a new system call to get SFrame information from the kernel. It could take several kernel releases to get everything right, he said. Sam James said that distributors are waiting for version 3 of the specification as well. 

#### Report from Paris

Thomas Schwinge reported that he had recently attended [Kernel Recipes](<https://kernel-recipes.org/en/2025/>) in Paris as a representative of the toolchain community, and had reported on the state of GCC there. The compiler, he said, is receiving commits at a rate of about 600/month — a rate that has remained essentially unchanged for the last two decades. He asked the group there how many were exclusively using LLVM to build their kernels, and only got a handful of responses; GCC is still relevant for kernel building, he said. 

Some participants at Kernel Recipes raised concerns about the aging of the GCC development community, but Schwinge answered that there are new contributors coming into the community. There was a request to be able to build for multiple architectures from a single build of the toolchain — something that LLVM can do, but GCC cannot. Boessenkool evidently has a project toward that end, and it is seen as an achievable objective. 

Schwinge concluded by noting that participants were happy for the ability to contribute to the toolchain with a developer's certificate of origin rather than a copyright assignment. They also liked the fact that new GCC releases don't tend to routinely break kernel builds, as happened more often in the past. There was a fair amount of interest in the state of `gccrs` as well. 

The session ran out of time and concluded at this point, though much of the same group reconvened to discuss BPF support after lunch (report forthcoming). There was general agreement that this type of meeting between the kernel and toolchain communities is valuable and should be repeated. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to this event.]
