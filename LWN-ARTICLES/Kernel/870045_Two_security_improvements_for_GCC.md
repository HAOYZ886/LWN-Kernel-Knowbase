---
title: Two security improvements for GCC
url: https://lwn.net/Articles/870045/
date: "September 24, 2021"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet September 24, 2021 LPC"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 24, 2021

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2021-Linux_Plumbers_Conference>)

It has often been said that the competition between the GCC and LLVM compilers is good for both of them. One place where that competition shows up is in the area of security features; if one compiler adds a way to harden programs, the other is likely to follow suit. [Qing Zhao's session](<https://linuxplumbersconf.org/event/11/contributions/1001/>) at the [2021 Linux Plumbers Conference](<https://linuxplumbersconf.org/>) told the story of how GCC successfully played catch-up for two security-related features that were of special interest to the kernel community. 

#### Call-used register wiping

Zhao started with a list of security features that kernel developers had been asking for, noting that the LLVM Clang compiler already had a number of them, but GCC did not. She has been working to fill in that gap, starting with the feature known as "call-used register wiping" — clearing the contents of registers used by a function before returning. There are a couple of reasons why one might want this feature in a compiler. 

The first of those is to frustrate [return-oriented programming (ROP)](<https://en.wikipedia.org/wiki/Return-oriented_programming>) attacks, which feature regularly in published [![\[Qing Zhao\]](https://static.lwn.net/images/conf/2021/lpc/QingZhao-sm.png)](<https://lwn.net/Articles/870065/>) exploits. A ROP attack works by chaining together a set of "gadgets" — code fragments that perform some useful (to the attacker) function followed by a return. If an attacker can place the right series of "return addresses" on the stack, they can string together a collection of gadgets and get the kernel to do just about anything that they want. 

ROP attacks must usually, sooner or later, call some other kernel function to carry out a needed task; the called function will look at the processor registers for its parameters. Making a ROP attack work thus requires getting the right values into those registers; clearing the registers at each function return can be highly effective at frustrating those attacks. It breaks the chain of gadgets that the attacker is trying to assemble. 

The other reason to clear registers on return, of course, is to prevent information leaks. It is often surprising to see what attackers can learn from whatever data may have been left in a CPU register. 

So clearing registers is good, but there is still the question of _which_ registers need clearing. If the objective is frustrating ROP attacks, clearing only the registers that are used for function parameters is sufficient. Protecting against information leaks, instead, requires clearing all of the registers used. A related question is whether registers should be zeroed or set to random values. For GCC, zero was seen as the safest choice, since it is the least likely to produce values that seem meaningful to other code. It also leads to a smaller and faster implementation. 

This functionality is part of the GCC 11 release, controlled by the `-fzero-call-used-regs=` compiler option, which has a number of possible values to control which registers should be cleared. There is also a new function attribute (`zero_call_used_regs`) that can be used to control register clearing for a specific function. The implementation is in the form of a new compiler pass that looks at all exit blocks, finds each return instruction, computes the set of registers to clear (which includes tracking which registers are actually used), and emits the instructions to actually perform the clearing. This functionality initially supported the x86 and Arm64 architectures; SPARC was added a bit later. 

Support for register clearing when compiling with GCC was [merged into the mainline kernel](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a82adfd5c7cb>) for the 5.15 release; the changelog notes that it reduces the number of usable ROP gadgets in the kernel by about 20%. 

#### Stack variable initialization

The C programming language famously specifies that automatic (stack) variables are not initialized by the compiler. If code uses such a variable before assigning a value to it, it will be working with garbage data that can lead to all kinds of problems. Erroneous outcomes are clearly one of those, but it gets worse; if an attacker can find a way to place a value on the stack where an automatic variable is allocated, they may well be able to compromise the system. If an uninitialized variable is used as a lock, the result can be uncontrolled race conditions. This is all worth avoiding. 

There are a number of tools around that can try to detect the use of uninitialized stack variables. Both GCC and Clang support the `-Wuninitialized` option, which causes warnings to be emitted at compile time, for example. Both compilers also have a `-fsanitize=` option to detect these usages at run time. Beyond the compilers, tools like [Valgrind](<https://www.valgrind.org/>) can be used to find uninitialized-variable usage. 

These tools are useful, but they have their limits, Zhao said. Static (compile-time) tools can only perform analysis within individual functions, which can require making assumptions about what other functions do. Their ability to detect problems with uninitialized array elements or values accessed via pointers is limited. So they miss problems while, at the same time, failing to prune out infeasible paths through the code and generating false-positive warnings. Dynamic (run-time) tools cannot cover all paths, so they will miss problems; they also impose a significant run-time overhead. 

Starting with the upcoming GCC 12 release, the `-ftrivial-auto-var-init=` option will control the automatic initialization of on-stack variables. Its default value, `uninitialized`, maintains the current behavior. If it is set to `pattern`, variables will be initialized to values that are likely to result in crashes if they are used; this option is intended for debugging use. Setting it to `zero`, instead, simply initializes all on-stack variables to zero; this option is for hardening production code. There is a new variable attribute (`uninitialized`) that can be used to mark variables that are deliberately not initialized. 

Regardless of the setting of this new option, the compiler will still issue warnings if `-Wuninitialized` is set. The idea behind this option is not to "fork the language", but to add an extra level of safety; code that fails to properly initialize variables should still be fixed. This work was committed to the GCC trunk in early September; there are some bugs still in need of fixing that should be taken care of soon. 

Zhao didn't talk about support for this feature in the kernel. Clang has had support for this option for a while, though, and the kernel can make use of it, so making use of GCC's support once it is available will be straightforward. That should help prevent whole classes of bugs, and may spell the beginning of the end for the [structleak GCC plugin](<https://lwn.net/Articles/712161/#structleak>) that is supported by the kernel now. While the development of these features was driven by a kernel wishlist, they should both prove useful well beyond the kernel context. 

The [video for this talk](<https://www.youtube.com/watch?v=wWz1j5XlTaI&list=PLVsQ_xZBEyN2c21jFUgqI2iMa094zXanH&index=31&t=2h11m>) is available on YouTube.
