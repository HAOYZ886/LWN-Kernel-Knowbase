---
title: Indirect branch tracking for Intel CPUs
url: https://lwn.net/Articles/889475/
date: "March 31, 2022"
category: "Releases-5.18; Security-Control-flow integrity"
author: "By Jonathan Corbet March 31, 2022"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 31, 2022

"Control-flow integrity" (CFI) is a set of technologies intended to prevent an attacker from redirecting a program's control flow and taking it over. One of the approaches taken by CFI is called "indirect branch tracking" (IBT); its purpose is to prevent an attacker from causing an indirect branch (a function call via a pointer variable, for example) to go to an unintended place. IBT for Intel processors has been under development for some time; after an abrupt turn, support for protecting the kernel with IBT has been merged for the upcoming 5.18 release. 

The kernel, like many C programs, makes extensive use of indirect branches. As a simple example, consider system calls; user space provides a number indicating which system call is required, and the kernel responds by looking up the appropriate function from a table (using that number) and calling that function via an indirect branch. Function pointers abound in the kernel; among other things, they are used to implement its vaguely object-oriented programming model. 

If an attacker is able to somehow corrupt a variable that is used for indirect branches, they may be able to redirect the kernel's execution flow to an arbitrary location. That could result in unintended function calls; on complex processors like x86, it is also possible to get interesting results by jumping into the middle of a multi-byte instruction. Exploit techniques like [return-oriented programming](<https://en.wikipedia.org/wiki/Return-oriented_programming>) and [jump-oriented programming](<https://developer.arm.com/documentation/102433/0100/Jump-oriented-programming>) depend on this kind of redirection. 

IBT is meant as a defense against jump-oriented programming; it works by trying to ensure that the target of every indirect branch is, in fact, intended to be reached that way. There are a number of approaches to IBT, each with its own advantages and disadvantages. For example, the kernel gained support for [a compiler-implemented IBT mechanism](<https://lwn.net/Articles/856514/>) during the 5.13 development cycle. In this mode, the compiler routes every indirect branch through a "jump table", ensuring that the target is not only meant to be reached by indirect branches, but that the prototype of the called function matches what the caller is expecting. This approach works, at the cost of a fair amount of compile-time and run-time overhead. 

#### Intel's IBT

The Intel IBT approach is rather simpler, but it has the advantage of being supported by the hardware and, as a result, being faster. If IBT is enabled, the CPU will ensure that every indirect branch lands on a special instruction (`endbr32` or `endbr64`), which executes as a no-op; if anything else is found, the processor will raise a control-protection (`#CP`) exception. Unlike the more complete scheme described above, IBT cannot ensure that the target of an indirect branch matches the caller's expectations, but it can ensure that the target was meant to be reached in this way. 

Turning on a mechanism like this will only work if every possible target of an indirect branch begins with one of the `endbr` instructions. For the most part, this task can be handled by the compiler; both GCC (as of GCC 9) and Clang (as of version 14) implement the `-fcf-protection=branch` option and will insert these instructions when it is present. That doesn't help with all of the assembly code in the kernel, though. So the bulk of the work (in terms of changesets) is devoted to adding `endbr` instructions wherever they seem to be needed. 

One other small complication comes about when the kernel calls into somebody else's code, which may not have been built with IBT in mind. The kernel does not call outside code often, but one big exception is the system's firmware, which must often be invoked to carry out specific functions. To be safe, the kernel makes a point of turning off IBT around calls into firmware. The current implementation also turns off IBT when giving control to user space. 

The need to add `endbr` instructions to all indirect jump targets sets a potential trap for the future; developers may add assembly functions and forget that instruction. If they do their testing without IBT enabled, the omission will not be noticed, and it may not pop up until some extremely inconvenient time after the faulty work has been merged. To prevent this eventuality, the kernel's `objtool` utility has been enhanced to check all indirect branches and ensure that all targets are appropriately annotated. 

With that checking in place, though, there's another step that can be taken: `objtool` can also make a list of all functions containing `endbr` instructions that can never be called via an indirect branch. Those functions do not need that annotation, and the kernel would be a little more secure without them. So the kernel build process takes that list from `objtool` and "seals" those functions by overwriting the `endbr` with a `nop4` instruction. That reduces the number of targets an attacker can still choose from when IBT is enabled. 

As Peter Zijlstra [pointed out](<https://lwn.net/ml/linux-kernel/YZyuPffZU5bOjzBQ@hirez.programming.kicks-ass.net/>), there is another, perhaps surprising advantage to removing the unneeded `endbr` instructions. The kernel limits the functions that are available to loadable modules, and proprietary modules are limited even further. It is a common technique for proprietary modules to look up the non-exported functions they need in the kernel's symbol table, then call them via an indirect branch, thus bypassing the kernel's limitations. But, with IBT enabled, any function lacking an `endbr` instruction will no longer be callable in this way. 

#### An indirect path to the mainline

The effort to get Intel IBT support into the Linux kernel has been ongoing for some time; the first patches implementing support (for user-space code rather than for the kernel) were [posted by Yu-cheng Yu](<https://lwn.net/ml/linux-kernel/20180607143855.3681-1-yu-cheng.yu@intel.com/>) in 2018. This work then seemingly became one of those [flying-Dutchman](<https://en.wikipedia.org/wiki/Flying_Dutchman>) patches that continually cross the mailing lists without ever managing to land in the mainline; [version 30](<https://lwn.net/ml/linux-kernel/20210830182221.3535-1-yu-cheng.yu@intel.com/>) was posted in August 2021 and seemed no closer to merging. A similar fate befell the user-space shadow-stack patches, which were recently [taken over by Rick Edgecombe](<https://lwn.net/Articles/885220/>) after many previous revisions. 

Late last year, Peter Zijlstra decided to create a separate Intel IBT implementation to protect the kernel itself; the first version was [posted last November](<https://lwn.net/ml/linux-kernel/20211122170301.764232470@infradead.org/>) after Zijlstra evidently ""hacked this up on Friday night / Saturday morning"". The work evolved quickly, and the [fourth revision](<https://lwn.net/ml/linux-kernel/20220308153011.021123062@infradead.org/>), posted in early March, is the code that was merged for 5.18. 

That is where things stand today. IBT is supported, for kernel code only, in Intel processors starting with the [Tiger Lake](<https://en.wikipedia.org/wiki/Tiger_Lake>) generation, which hit the market in late 2020. It is not a perfect tool, but it will raise the bar for attackers on systems where it is present and enabled. Meanwhile, it is not clear when (or whether) user-space support will find its way into the kernel; many of the 30 revisions posted so far have received no comments at all.
