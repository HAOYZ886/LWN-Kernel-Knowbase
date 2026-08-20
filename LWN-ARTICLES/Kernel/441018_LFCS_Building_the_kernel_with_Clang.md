---
title: "LFCS: Building the kernel with Clang"
url: https://lwn.net/Articles/441018/
date: "May 4, 2011"
category: "Development tools-LLVM"
author: "By Jake Edge May 4, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 4, 2011

Back in October, Bryce Lelbach [announced](<https://lwn.net/Articles/411654/rss>) that he (and others) had built a working Linux kernel using (mostly) Clang, the LLVM-based C compiler. At the Linux Foundation Collaboration Summit (LFCS) back in April, Lelbach gave a talk about the progress that had been made, and the work still to be done, for the [LLVM Linux (LLL) project](<https://github.com/lll-project>). That talk, along with the rest of the LLVM track, was quite interesting, and once again showed that having two (or more) "competing" projects is generally beneficial to both. 

#### Why build Linux with Clang?

Lelbach started off describing the reasons behind the decision to try to build Linux with Clang, most of which centered around the diagnostics that the compiler produces. The Clang static analyzer has the ability to show ""what the compiler sees when it's looking at your code"", he said. He thought that a huge codebase like Linux could benefit from that kind of analysis. 

In fact, the Clang diagnostics were quite useful when he was building the Broadcom wireless driver for his MacBook, he said. Clang doesn't forget things, so it can show macros before their expansion, typedefs, and so on. It also shows the line in the source code with a caret pointing to the offending code, along with "fixit hints". Those hints can be automatically applied to the source code to fix the problem in question. 

The project got a 2.6.36-based kernel running back in October, and now has working kernels based on .37 and .38. Neither Xen nor KVM worked at the time of the talk and Xen won't even compile, though KVM is said to work now. More than 90% of the drivers in the kernel will at least compile, and many will work. Some out-of-tree binary drivers (Broadcom, NVIDIA) will work as well. SMP versions of the kernel for both 32 and 64-bit x86 platforms are now working, though some of the code needs to be patched in order to build correctly. 

#### Things that don't work

The integrated assembler (IA) for Clang does not have support for generating "real mode" code using `.code16gcc` directives, so the Linux boot code cannot be built using IA. There is a ""nasty pile"" of real mode code required to boot on x86, Lelbach said. IA is the default for recent versions of Clang, but using the GNU Assembler (gas) was required for the boot code. Adding support for an LLVM x86-16 backend is the right approach, he said, and LLVM project members in the audience agreed that it was something that could be added to IA. 

The ""vast majority of GCC extensions are supported"" by Clang, even those which are not documented, which makes compiling the kernel much easier. Things like inline assembly, the `__attribute__` and `__builtin__` syntax, and so on, all just work. He expected that there might be problems with inline assembly, but that has not proven to be the case. Clang defaults to the C99 standard, though, so the `gnu89` standard needs to be specified to build the kernel. 

There are some GCC extensions that aren't implemented, however, including explicit register variables. That lack blocks Xen and some user-space libraries (like glibc) from compiling. There are also some ""intentionally unsupported extensions"", including local/nested functions, which is only used in a Thinkpad driver. A bigger problem is that Clang lacks support for variable-length arrays in structures (VLAIS). A declaration like: 

```
void f (int i) {
            struct foo_t {
                char a[i];
            } foo;
        }
```

cannot be compiled in Clang, though declarations like: 

```
void f (int i) {
            char foo[i];
        }
```

are perfectly acceptable. Code like the former is used in the iptables code, the kernel hashing (HMAC) routines, and some drivers. Those parts have to be [patched](<https://github.com/lll-project/kernel/commit/d828bfdcbaf01d4a0abae5642c42c969b7ceb980>) in order to be built, he said. Once again, someone from the audience piped up to say that support for VLAIS could be added as long as the patches were not ""wildly invasive"". The LLL folks ""prefer adding things to Clang rather than patching the kernel"", Lelbach said. 

That led to a question about whether the project was pushing any of its patches upstream to the kernel. Lelbach said that the PaX team (who is another LLL developer) had submitted a few, but that those were rejected; ""after three, we stopped"" submitting them. Part of the problem is that the patches are not ready for inclusion because there is a lack developer time to get them into shape. As an audience member noted, though, the kernel folks are quick to take any patches that fix bugs found by Clang. 

#### Code generation and optimization problems

There are several code generation and optimization options for GCC that aren't supported by Clang. One of those is `-mregparm` that governs the number of registers used to pass integer arguments. That means calls to functions like `memcpy()` are generated that ignore the custom calling conventions. 

Also, `-fcall-saved-reg` is not supported by Clang and that affects the uses of the `ALTERNATIVE()` macro in the kernel, which chooses between assembly instructions depending on the processor model. For some of the `__arch_hweight*()` implementations `ALTERNATIVE()` buries the actual function call inside assembly code, so Clang doesn't know about it. That means that the generated code is not saving all of the registers that it needs to, so uses of `ALTERNATIVE()` are commented out and a normal call to the function is used instead. 

Another problem is with `-pg`, which enables instrumentation code for function calls in GCC, and is used when building Ftrace. For inline functions, the calls to `mcount()` get added multiple times, both when the code is generated and when it is expanded inline. The `no_instrument_function` attribute is not properly propagated to inline functions, he said. 

The final problem that Lelbach mentioned is the `-fno-optimize-sibling-calls` flag that is not supported by Clang. The flag disables tail call elimination, and the kernel introspection code (like Ftrace) assumes specific stack depths in various places. Because Clang doesn't support the flag, code which walks the call stack can end up dereferencing user-space pointers, which leads to runtime crashes. This was worked around by defining `HAVE_ARCH_CALLER_ADDR` for x86 and defining `CALLER_ADDR[1-6]` as dummy values, effectively disabling the stack backtracing. 

It is not just Lelbach who is working LLL, and he noted that the PaX team, Alp Toker, and Torok Edwin have all contributed, along with various Clang/LLVM and Linux kernel hackers. There are plans to create a mailing list for the project and the beginnings of a [wiki](<https://github.com/lll-project/kernel/wiki/>) are taking shape. Overall, it's an interesting project that will likely end up helping to find bugs in the kernel while discovering features that could or should be supported by LLVM/Clang. 

[ Thanks to Bryce Lelbach, PaX team, and Torok Edwin for filling in holes in my notes. ]
