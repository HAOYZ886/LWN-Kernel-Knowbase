---
title: "Control-flow integrity for the kernel"
url: https://lwn.net/Articles/810077/
date: "January 22, 2020"
category: "Security-Control-flow integrity"
author: "By Jake Edge January 22, 2020 LCA"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
January 22, 2020

* * *

[LCA](<https://lwn.net/Archives/ConferenceByYear/#2020-linux.conf.au>)

[Control-flow integrity](<https://en.wikipedia.org/wiki/Control-flow_integrity>) (CFI) is a technique used to reduce the ability to redirect the execution of a program's code in attacker-specified ways. The Clang compiler has some features that can assist in maintaining control-flow integrity, which have been applied to the Android kernel. Kees Cook gave a talk about CFI for the Linux kernel at the recently concluded [linux.conf.au](<https://linux.conf.au/>) in Gold Coast, Australia. 

Cook said that he thinks about CFI as a way to reduce the attack, or exploit, surface of the kernel. Most compromises of the kernel involve an attacker gaining execution control, typically using some kind of write flaw to change system memory. These write flaws come in many flavors, generally with some restrictions (e.g. can only write a single zero or only a set of fixed byte values), but in the worst case, they can be a "write anything anywhere at any time" flaw. The latter, thankfully, is relatively rare. 

#### Background

A historical attack model was to simply write to regions of the running code, which was used by "ancient rootkits and [...] anti-virus software, which were indistinguishable from rootkits". For that kind of attack to work, the target must have both executable and writable memory permissions. User space can write and execute anywhere in its address space, but it cannot write into that portion reserved for kernel space. A write flaw in the kernel would effectively allow user space to write into kernel memory, however. 

[ ![\[Kees Cook\]](https://static.lwn.net/images/2020/lca-cook-sm.jpg) ](<https://lwn.net/Articles/810083/>)

That led to the no-execute (NX) bit being used to disallow execution in the data portions of the kernel's address space; the portions of the address space for kernel text modules were still executable, though. That led to the idea of making those parts read-only (RO) so that they were not both writable and executable, which is what was needed for an exploit of that sort. 

But that still left all of user space that could be both written to and executed from via a kernel exploit. That led to using the Intel supervisor mode execution prevention (SMEP) and ARM privileged execute never (PXN) features to restrict the kernel from executing user-space memory (while in kernel mode). That closed off any place that would be both writable and executable. All of those protections make up "the last century of defenses", he said. 

So, thinking as an attacker, what can be done within those constraints? One possibility is to use addresses stored in memory; by overwriting those addresses, an attacker can control which code is executed. The most common place to do so is by manipulating return addresses on the stack, which is what [return-oriented programming](<https://en.wikipedia.org/wiki/Return-oriented_programming>) (ROP) attacks do. 

Function pointers are used for indirect function calls, which are different than direct function calls because the address of the call site is not stored in the (non-writable) kernel text. Instead, the address for the call site is fetched from memory, placed into a register, and the call is made via that value. If an attacker can change the memory, they can control where the call actually ends up going. That is the "forward edge" of an indirect call, while the return address on the stack is the "backward edge" of the call. Either can be used by an attacker to redirect the code flow. 

The writable function pointers can only exist in the kernel's heap and stack due to the earlier tightening of the access to the rest of memory. Function pointers can be stored in the heap or on the stack. It turns out that making the stack read-only "makes it very hard to use", Cook said with a chuckle. If an attacker can overwrite one of these two edges, they can call any executable byte in the kernel, "which is a gigantic exploitation surface". 

#### Enter CFI

The goal behind CFI is to try to ensure that indirect calls go to the expected addresses and that the return addresses are not changed. For the forward edge, indirect function pointers need to be validated before the call is made. Function pointers can be categorized into "classes" based on their prototype and return value; doing so allows restricting indirect calls to functions in the same class as the original. Hardware CFI protections, such as Arm's [branch target identification](<https://lwn.net/Articles/804982/>) (BTI), have poor granularity since they only force indirect calls to go to the start of a function, which is not all that much protection, he said. 

The Clang compiler can ensure that calls are only made to functions in the same class, which "narrows things quite a bit". In order to do so, though, Clang needs to use [link-time optimization](<https://en.wikipedia.org/wiki/Interprocedural_optimization#WPO_and_LTO>) (LTO) to get visibility across the whole kernel code base. Functions in the same class are collected into jump tables that are checked at runtime before an indirect call is made. Cook's [slides [PDF]](<https://outflux.net/slides/2020/lca/cfi.pdf>) have detailed examples of how that all works. 

There is sometimes a bit of a performance hit from doing the validation, he said, but it is "not terrible". The Clang method works well enough, but there are other implementation ideas out there. The [Reuse Attack Protector [PDF]](<https://pax.grsecurity.net/docs/PaXTeam-H2HC15-RAP-RIP-ROP.pdf>) (RAP) technique from PaX Team is a "clever idea" that uses hash values in the kernel text at function entry and exits, then checks those values by reading them from the kernel text. It is not compatible with execute-only memory (i.e. memory that cannot be read, just executed from), which is coming. 

Another mechanism is the (as yet unreleased) [kCFI [PDF]](<https://www.blackhat.com/docs/asia-17/materials/asia-17-Moreira-Drop-The-Rop-Fine-Grained-Control-Flow-Integrity-For-The-Linux-Kernel-wp.pdf>); it makes a second pass to determine which functions are legitimately reachable from call sites. The idea is to further reduce the exploitability surface of having multiple functions all in the same class. For example, there are many kernel functions with a "`void foo(void)`" signature. The Android kernel team [looked at the targets for indirect calls](<https://android-developers.googleblog.com/2018/10/control-flow-integrity-in-android-kernel.html>) and found that 55% have five or less targets—but 7% have more than 100. 

#### Backward edge

For the backward edge, some kind of trusted stack (e.g. a shadow call stack) to contain the return addresses is needed. It looks like that is best done in hardware (e.g. Intel's [Control-Flow Enforcement Technology](<https://lwn.net/Articles/758245/>) (CET) and Arm's [pointer authentication](<https://lwn.net/Articles/718888/>)), but the hardware support is not there yet, so software versions are needed. 

The x86 version of shadow stacks in Clang was slow and there were some race conditions, so it was removed, but the picture for Arm is better. A single register is reserved for Arm processors to be used for shadow-stack operations; its value is not stored on the regular stack to avoid revealing the location of the shadow stack. Only the return addresses are pushed onto the shadow stack, but they are also still pushed onto the regular stack so that call-stack unwinders and the like will continue to work. Before the return from a function, the shadow stack value is popped and used. 

The need to keep the location of the shadow stack secret makes that solution less than perfect; the hardware solutions will be much better. For Intel CET, there is no change needed in the generated assembly code, though the shadow stack will need to be set up by the language runtime; calls and returns will automatically use the otherwise-read-only stack. For Arm pointer authentication, addresses will need to be signed using the `-msign-return-address` option (for both Clang and GCC) and two new instructions (`paciasp` and `autiasp`) will be needed at entry and before return instructions. 

The Pixel 3 and later Android phones are using CFI; forward-edge protection was added in Q3 2018 and backward-edge protection in Q3 2019. The code is also in the Android common kernel. The Android compatibility definition strongly recommends the use of these features; strong recommendations typically become requirements for the next Android release, he noted. 

There were a number of "gotchas" in trying to make this all work. For one thing, using LTO requires a huge amount of memory and CPU, which resulted in "massive linking times". There is a ThinLTO mode for Clang, however, which does less analysis but, once a few bugs were fixed, worked for CFI; "now the linking times are not egregious, just slow". Clang would only build the jump tables for C functions, but the kernel has lots of assembly language functions (e.g. cryptographic algorithms), so Clang needed to be extended to build jump-table entries for all `extern` functions as well. 

Beyond that, relative addresses that are calculated as a delta from the actual function address, which are used by the [kernel exception tables](<https://www.kernel.org/doc/html/latest/x86/exception-tables.html>) for user-space accesses, would fail because the address did not appear in the jump tables. The exception tables are hard-coded, however, so the CFI checks were simply disabled when jumping to those relative addresses. Ftrace uses linker aliases in a way that confused the CFI checks, but different linker aliases can be added to fix them. In addition, [kernel-page-table isolation](<https://lwn.net/Articles/752621/>) (KPTI), which was added to deal with Meltdown, ensures that only a small entry stub for the kernel was mapped for user-space programs, but the jump tables are also needed by that entry stub, so they had to be added to the regions mapped under KPTI. 

#### Upstreaming

He is cautiously optimistic ("fingers crossed") that all of the needed changes in Clang to build Linux will be in the upcoming LLVM 10 release. While it is not related to CFI, the "asm goto" feature was needed in Clang in order to build x86 kernels; he wants CFI to be available for as many Linux targets as possible and "it turns out that x86 has a rather large installed base". 

On the kernel side, the Clang shadow-call-stack support is separable from other dependencies and is expected to be added in the 5.6 merge window. In order to support forward-edge CFI, the function prototypes need to be correct for all of the indirect call sites, but there were a numbers of places in the kernel where that was not true. At this point, Arm has all of those fixed, but there is one remaining patch to correct a prototype for x86. 

Adding LTO support is mostly just changes to the build scripts; it is all working, but it needs to get upstream. There were a lot of small details that needed to be worked out and agreed upon; he is hoping that all of those are settled now so that LTO can be merged. The actual forward-edge CFI support requires LTO; he is hoping that the CFI piece "is entirely uncontroversial, which is never true". In any case, he is optimistic for inclusion in the 5.7 or 5.8 kernel. 

There are some "do it yourself" instructions and links near the end of Cook's slides. All of the outstanding 50 or so patches are in a [GitHub repository](<https://github.com/samitolvanen/linux/tree/clang-cfi>). One thing to keep in mind is that even when simply configuring the kernel, the build scripts need to know what compiler and linker are being used, so it is necessary to specify Clang and the LLVM linker on the `make` command line ("`CC=clang LD=ld.lld`"). 

There are two modes for CFI; `CONFIG_CFI_PERMISSIVE` is effectively the debug version of CFI, it will simply warn of CFI problems and continue running. If it is not set to permissive mode, either the kernel will panic or the thread will be killed depending on the type of CFI failure and "how it is set up internally". That is only for forward-edge CFI failures, however; the backward-edge failures are not reported currently as any changed return address is just ignored and the correct value is used from the shadow stack. That is something he would like to see changed in future versions so that some kind of warning would be provided. 

In answer to a question, Cook said that the forward-edge CFI protection works fine with [retpolines](<https://support.google.com/faqs/answer/7625886>) (which is a Spectre mitigation), but he turned them off for his examples as it just complicates the picture. Hardware-based forward-edge CFI might make retpolines impossible, but he is hopeful that retpolines will become unnecessary well before some new scheme for CFI makes its appearance. While ROP attacks target places other than function entry, most other attacks are calling functions "normally", so the restriction that is provided by today's hardware CFI is not terribly useful, he said. 

Interested readers may also wish to view the [YouTube video](<https://www.youtube.com/watch?v=0Bj6W7qrOOI>) of the talk. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to Gold Coast for linux.conf.au.]
