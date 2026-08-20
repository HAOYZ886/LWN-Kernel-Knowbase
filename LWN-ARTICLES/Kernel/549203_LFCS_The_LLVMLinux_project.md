---
title: "LFCS: The LLVMLinux project"
url: https://lwn.net/Articles/549203/
date: "May 7, 2013"
category: "Development tools-LLVM"
author: "By Jake Edge May 7, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 7, 2013

The [Linux Foundation Collaboration Summit](<http://events.linuxfoundation.org/events/collaboration-summit>) (LFCS) seems to be a likely venue for an update on the status of building the kernel with Clang/LLVM. Both [in 2011](<https://lwn.net/Articles/441018/>) and [2012](<https://lwn.net/Articles/493312/>), we covered those updates. LFCS 2013 continued the trend as [LLVMLinux project](<http://llvm.linuxfoundation.org/index.php/Main_Page>) lead Behan Webster presented the status and plans for the project at LFCS. The gathering lived up to its name as well, since two problems faced by the project were solved through collaboration at the summit. 

![\[LLVMLinux logo\]](https://static.lwn.net/images/2013/lfcs-llvmlinux.png)

Webster is a computer engineer who has worked in a lot of different industries: automotive, telecommunications, embedded systems, etc. For 20 years or so, he has worked on Linux, using it either as a development environment or as the operating system of a shipping product. He is the project lead for the LLVMLinux project, which is an effort to get the Linux kernel to build with Clang and LLVM. 

[LLVM](<http://llvm.org/>) is a set of libraries that can be used to build tools, he said. Those tools could be compilers, linkers, or JITs, or they could be tools for source code analysis, metadata extraction, code refactoring, and so forth. Most of the latter kinds of tools don't directly relate to LLVMLinux, but the toolchain for building C, C++, and Objective C programs does. The toolchain starts with [Clang](<http://clang.llvm.org/>), the frontend for C-family programs, but there are other pieces as well, including the [compiler-rt](<http://compiler-rt.llvm.org/>) library, [lld](<http://lld.llvm.org/>) and [MCLinker](<http://code.google.com/p/mclinker/>) linkers, [checker](<http://clang-analyzer.llvm.org/>) static analyzer, [LLDB](<http://lldb.llvm.org/>) debugger, and others. 

#### Why?

There are a number of reasons, beyond just simply having a choice, that make compiling Linux with LLVM a useful endeavor. For one thing, it compiles faster, which is important to reduce the time of the code-compile-run-debug cycle. LLVM is a fast-moving project, Webster said, and it is "amazing how fast Clang has caught up" to GCC in some areas. It generates faster code than GCC in some areas--slower code in others. 

LLVM is already being used in many different domains, like audio and video in such projects as [llvmpipe](<http://www.mesa3d.org/llvmpipe.html>), [CUDA](<https://developer.nvidia.com/what-cuda>), and [Renderscript](<http://developer.android.com/guide/topics/renderscript/index.html>). There is an advantage to having a single toolchain used in all of these different domains as compiler extensions only need to be written once and can then apply in lots of different places. LLVM also has a different license--BSD-style--which is not better or worse than others, Webster said, but it allows tool vendors to do things with LLVM that they couldn't with GCC. LLVM is also well-supported by a large group of full-time developers. 

The static analyzer for LLVM is "amazing", though it does not yet work on the kernel code. This tool has traditionally done better than other, similar tools. One of the headline features is "fix-it hints", which are suggestions for fixing small, localized problems in the source; a similar feature now appears in GCC 4.8. That is one example where we are seeing "more and better" development from both projects because of the competition between them, Webster said. 

The Google compiler team has built a tool around LLVM that shows a novel use of the compiler toolkit. The tool looks for common problems that appear as certain patterns in the LLVM intermediate representation and can map them back to the C++ code to show the programmer where they made an error. This shows things that you can do that we haven't seen before, he said. 

Beyond Android's use of LLVM for Renderscript and Gallium3D's llvmpipe driver, there are also [distributions](<https://lwn.net/Articles/487359/>) looking at building all or parts of their repositories using LLVM. For Debian, Sylvestre Ledru has been building the repository using Clang. His most recent [results](<http://sylvestre.ledru.info/blog/sylvestre/2013/02/06/rebuild_of_debian_using_clang_3_2>) show that there are more failures than before, Webster said, because new versions of Clang are more strict. Gentoo is [looking at support for LLVM/Clang](<https://bugs.gentoo.org/show_bug.cgi?id=408963>), as is [FreeBSD](<https://wiki.freebsd.org/BuildingFreeBSDWithClang>). 

#### LLVMLinux

The goal of the LLVMLinux project is to get to the point where the kernel will build with LLVM. Ultimately, that means that LLVM would have everything it needs to build the kernel, while the kernel would get any changes it requires so that it can be built with LLVM. 

To that end, the project has a [Git repository](<http://git.linuxfoundation.org/llvmlinux.git>) for its build framework that contains scripts and patches to build the kernel. The scripts fetch, patch, and build various pieces, like LLVM/Clang, toolchains for cross assembly and linking, the kernel, and QEMU. 

Several cross-toolchains are supported, including Codesourcery (which is the default), Linaro/Ubuntu, and Android. Those are necessary for the GNU assembler and linker, as those pieces for LLVM are not yet mature enough to use for building the kernel. There is support in the project's tree for various targets, including Versatile Express, x86-64, Raspberry Pi, and Nexus 7\. There are other targets in progress, including the Galaxy S3 and BeagleBone. 

The project is using [Buildbot](<http://buildbot.net/>) for continuous integration. Any time there is a commit in the project, LLVM, or kernel repositories, a full build is kicked off. In addition, the [Linux Test Project](<http://ltp.sourceforge.net/>) suite is run nightly using QEMU for the Versatile Express. 

#### Problem areas

Getting to this point has been "challenging", Webster said, as there are a number of difficulties the project has run into. To start with, LLVM's integrated assembler (IA) can't be used on kernel code because it doesn't handle the format used by the kernel's assembly code. In addition, IA does not handle 16-bit code. Building would be faster using IA, he said, but that's just not possible right now, which results in a dependency on the GNU toolchain. 

The Linux code is GCC-specific in a number of ways. GCC (and thus the kernel) conforms to the gnu89 standard, while Clang conforms to gnu99 (which is "essentially C99" with some GCC additions). Webster thinks that a future version of GCC will move to C99, which will help. Kernel developers have long said that the standard is insufficient, so the kernel code goes beyond it in various ways. But, the standards have largely caught up, he said, though there are still some "notable exceptions". It is almost as if the Linux kernel has been driving the C standards. 

Beyond that, there are some GCC flags that the kernel uses which are not supported by Clang. Some of the built-in functions are very different between Clang and GCC. Another problem area is Kbuild, which is also GCC-specific. In particular, unsupported flags cause different return values from GCC and Clang. In both cases, a warning is issued, but the different return code causes problems. 

There are several GCC flags that are not supported by Clang, including `-fconserve-stack`, `-fdelete-null-pointer-checks`, `-fno-inline-functions-called-once`, and `-mno-thumb-interwork`. More details on these can be found in the [slides [ODP]](<http://www.linuxplumbersconf.org/2012/wp-content/uploads/2012/09/2012-LPC-LLVMLinux-bw2.odp>) from Webster and Mark Charlebois's 2012 Linux Plumbers Conference talk. 

There are a handful of GCC language extensions that will never be part of LLVM/Clang, he said, including variable-length arrays in structures (VLAIS). Zero-length arrays at the end of structures are supported, as are variable-length arrays outside of structures, but constructs like: 

```
struct foo {
            char a[n];
        } bar;
```

are not allowed by C99 or the more recent C11. The LLVM developers are not inclined to add VLAIS, as it makes the compiler harder to maintain, so LLVMLinux has to try to convince kernel developers to remove them. Currently, iptables, cryptographic hashing (HMAC), and a few other places use VLAIS and the maintainers like the code the way it is, Webster said. But the project has been running some tests on alternatives and getting the same or slightly better performance by switching away from VLAIS. 

A member of the audience asked if it made sense to fork Clang to add VLAIS support, but Webster downplayed that option. All of the Clang developers are of a single mind about this particular feature, he said, though they would be supportive of a fork if someone decided to take that path. In the end, it comes down to changing "dozens of lines" of kernel code versus a "significant architectural change" in the LLVM toolchain. Turning the arrays into pointers is an easy way to fix the problem, but some maintainers don't want to take changes just to support a different compiler, he said, in answer to follow-on questions. 

Nested functions are another problem area, but they are used infrequently. In general, when patches come in that have nested functions, the kernel developers have pushed back and asked for a rewrite. However, the Thinkpad ACPI driver still has nested functions, though Webster sent a patch to change that. He has not yet heard back, but it is just a rearrangement of the code, without any functional change. 

Neither of those two features are in the C standard, nor are they used very frequently in the kernel. It would be relatively easy to not require them. The other extensions used by the kernel are "innocuous and simple", he said. For example, Clang does not support variables explicitly assigned to specific CPU registers, but he would like to see that support get added. 

A recently discovered bug in Clang's C preprocessor is behind a problem with section reference mismatches. The attributes being used to put `__init` and `__exit` functions into different ELF sections are being mishandled. Webster "firmly believes" that the kernel pushes the tools harder than any other code base, and this is one example where that pushing has found bugs. It is probably just a corner case that had not been tested (or exercised by other code built with Clang), resulting in various "section reference mismatch" and "merged global" messages (as well as kernel module loading problems on x86) when the kernel was linked. 

Webster then mentioned `__builtin_const_p()`, which GCC makes available to test whether the argument is a constant value, but LLVM cannot support as it cannot really make that distinction. At that point, Steven Rostedt started poking around in the kernel code, asking the occasional question. In a post-summit email, Webster said that Rostedt tracked the use of `__builtin_const_p()` down to a particular commit in `mm/slab.c` made by Christoph Lameter, who was also present at the summit. Within 24 hours, Rostedt and Lameter worked out a replacement; Rostedt was even able to run it by Andrew Morton, who agreed in principle to the change. 

#### Status

For LLVM, all required patches from the project are now upstream. There are still some outstanding issues, including the section reference mismatch problem above. He believes that Clang 3.3 will mostly work out of the box for building kernels with the LLVMLinux patches. 

Another outstanding issue is a segmentation fault in Clang when it is compiling `arch/arm/mm/context.c`. That problem seemed to be related to using atomic_64 operations along with nearby inline assembly code and Webster was having difficulty creating a simple test case. After the talk, another attendee, Konstantin Serebryany, suggested using [creduce](<http://embed.cs.utah.edu/creduce/>), which is an LLVM-based tool that can bisect code to try to find a minimal subset that reproduces a problem. By the time Webster started figuring out the tool later that day, Serebryany had attached a minimal test case to the [bug report](<http://llvm.org/bugs/show_bug.cgi?id=15676>). Within another few days, it was fixed in the LLVM tree. In his email, Webster paraphrased the famous saying by noting that LFCS showed him that "the right eyes make bugs shallow". 

There are still some kernel patches that need to be pushed upstream as well. Kbuild support, removing VLAIS use, handling `__builtin_const_p()`, and so on, all need solutions either in the upstream kernel or elsewhere. 

It is not strictly necessary, but he would like to use the LLVM integrated assembler for its speed, too. Getting the checker static analyzer to run on the kernel would be useful as well. 

Anyone wanting to help will find a variety of ways to do so, Webster said. Helping to push patches upstream or work on unsupported features is one way. In addition, LLVMLinux is just working for x86 and ARM right now, adding MIPS and other architectures would be nice. As might be guessed, interested folks will find information on mailing lists, IRC, and more on the project's web page.
