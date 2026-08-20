---
title: "Per-system-call kernel-stack offset randomization"
url: https://lwn.net/Articles/816085/
date: "March 27, 2020"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet March 27, 2020"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 27, 2020

In recent years, the kernel has (finally) upped its game when it comes to hardening. It is rather harder to compromise a running kernel than it used to be. But "rather harder" is relative: attackers still manage to find ways to exploit kernel bugs. One piece of information that can be helpful to attackers is the location of the kernel stack; [this patch set](<https://lwn.net/ml/linux-kernel/20200324203231.64324-1-keescook@chromium.org/>) from Kees Cook and Elena Reshetova may soon make that information harder to come by and nearly useless in any case. 

The kernel stack will always be an attractive target. It typically contains no end of useful information that can be used, for example, to find the location of other kernel data structures. If it can be written to, it can be used for [return-oriented programming](<https://en.wikipedia.org/wiki/Return-oriented_programming>) attacks. Many exploits seen in the wild (Cook mentioned [this video4linux exploit](<https://a13xp0p0v.github.io/2020/02/15/CVE-2019-18683.html>) as an example) depend on locating the kernel stack as part of the sequence of steps to take over a running system. 

In current kernels, the kernel stack is allocated from the `vmalloc()` area at process creation time. Among other things, this approach makes the location of any given process's kernel stack hard to guess, since it depends on the state of the memory allocator at the time of its creation. Once the stack has been allocated, though, its location remains fixed for as long as the process runs. So if an attacker can figure out where the kernel stack for a target process is, that information can be used for as long as that process lives. 

As it turns out, there are a number of ways for an attacker to do that. Despite extensive cleanup work, there are still numerous kernel messages that will expose addresses of data structures, including the stack, in the kernel log. There are also attacks using [`ptrace()`](<http://man7.org/linux/man-pages/man2/ptrace.2.html>) and cache timing that can be used to locate the stack. So the protection offered by an uncertain stack location is not as strong as one might like it to be. 

Cook and Reshetova's patch set (which is inspired by the [PaX RANDKSTACK feature](<https://pax.grsecurity.net/docs/randkstack.txt>), though the implementation is different) addresses this problem by changing a process's kernel stack offset every time that process makes a system call. Specifically, it modifies the system-call entry code so that the following sequence of events happens: 

  * The `pt_regs` structure, containing the state of the processor registers, is pushed onto the base of the stack, just like it is done in current kernels. 
  * A call to `alloca()` is made with a random value. This has the effect of "allocating" a random amount of memory on the stack, which is really just a matter of moving the stack pointer down by that amount. 
  * The system call proceeds with its stack pointer in the now randomized location. 

In other words, the kernel stack itself doesn't move, but the actual stack contents shift around and are located differently for every system call. That makes any attack that depends on placing data at a specific location in the stack likely to fail; even if the attacker succeeds in figuring out where the stack is to be found, they won't know exactly where any given system call will place its data on that stack. 

Pushing the `pt_regs` structure before applying the randomization is important. The `ptrace()` attack mentioned above can be used to locate this structure (and thus the kernel stack); if it were located after the offset is applied, such attacks would thus reveal the offset. 

Currently, the randomization amount is obtained by reading some low-order bits from the CPUs time-stamp counter. Cook notes that other, more robust sources of entropy can be added in the future, but he doesn't think that needs to be figured out before the current patches can be considered. There are currently five bits of entropy applied to the stack offset on 64-bit systems, and six bits on 32-bit systems. That is not a huge amount of entropy, but it is enough that any attack that depends on precise kernel-stack locations will probably fail — and generate a kernel oops — on the first few tries. More entropy can be added, at the cost of wasting more stack space. 

With this feature in use, Cook measured the overhead as being about 0.9% on a no-op system call; it would clearly be less on any system call that does real work. But for people who don't want to pay even that cost, there is a static label to turn the randomization off. 

The end result is a relatively simple mechanism to further harden the kernel against attack. Cook [noted](<https://lwn.net/ml/linux-kernel/202003260932.510967DD@keescook/>) that it's not perfect, adding that ""most things can't be given the existing kernel design trade-offs"". If other developers agree, per-system-call stack offset randomization is likely to find its way into the mainline kernel's arsenal of hardening techniques.
