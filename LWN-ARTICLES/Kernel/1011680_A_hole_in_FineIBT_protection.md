---
title: A hole in FineIBT protection
url: https://lwn.net/Articles/1011680/
date: "February 27, 2025"
category: "Security-Control-flow integrity"
author: "By Jonathan Corbet February 27, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 27, 2025

Intel's [indirect branch tracking (IBT)](<https://en.wikipedia.org/wiki/Indirect_branch_tracking>) is a hardware-implemented control-flow-integrity mechanism that makes it harder for an attacker to gain control of the system by way of a corrupted indirect branch. FineIBT is a software extension to IBT that is meant to improve its protection. Recently, though, Jennifer Miller [reported](<https://lwn.net/ml/all/Z60NwR4w/28Z7XUa@ubun>) a novel way to bypass FineIBT by taking advantage of how the kernel's system-call entry point is constructed. In response, Peter Zijlstra is working on [some FineIBT](<https://lwn.net/ml/all/20250224123703.843199044@infradead.org>) enhancements to close that hole and make IBT more secure in general. 

The kernel (like many other programs) makes extensive use of indirect branches, typically in the form of a function call using a pointer value that is determined at run time. These indirect calls have always been an attractive target for attackers; if that pointer can be set to an attacker-controlled value, the call can be sent to an arbitrary location. Usually, that is all that is needed to gain control over the system. In a body of code as large as the kernel, there will surely be an instruction sequence that performs some useful operation for an attacker when it is invoked in an unexpected way. 

IBT implements forward-edge control-flow integrity by blocking indirect calls to arbitrary locations. It works by requiring that the target of an indirect call be a special instruction, either `endbr32` or `endbr64` (collectively referred to as `endbr`), that serves as a marker indicating a legitimate call target. IBT greatly reduces the number of places an indirect call can go; instead of anywhere in the program (including in the middle of a multi-byte instruction), calls can only go to actual function entry points. 

This restriction improves security, but only by so much. There are still a lot of functions in a program like the kernel, and much mayhem can be created by redirecting the control flow to an unexpected function. The protection would be much tighter if IBT could ensure that an indirect call lands on one of the intended targets, rather than on any function. FineIBT, which was described in [this paper](<https://arxiv.org/abs/2303.16353>) in 2023, is a step in that direction. In the kernel's implementation, every indirect call is modified to first load a special hash value into a specific register. The called function, immediately after the `endbr` instruction, will compare that hash against the expected value; if the two do not match, execution is aborted. The hash is generated from the prototype of the called function, but is then perturbed at boot time so that the hashes used in any given running system are different (and hopefully unknown to attackers). 

FineIBT was merged for the 6.2 kernel and has, hopefully, made life a bit harder for attackers ever since. As Miller has demonstrated, though, that protection is not absolute. In this case, the way around FineIBT takes advantage of one special assembly-language function within the kernel, [`entry_SYSCALL_64()`](<https://elixir.bootlin.com/linux/v6.13.4/source/arch/x86/entry/entry_64.S#L49>), which is called by the CPU (on x86_64 systems) when user space makes a system call. Looking at the code, one can see that it begins, as expected, with an `endbr` instruction; IBT requires that, even in response to a system-call trap. 

The following instruction, though, is not the usual FineIBT hash validation, since this function will not be called from within the kernel. Instead, it is `swapgs`, which exchanges the contents of the processor's GS-segment base register with the contents of a special model-specific register (MSR). This instruction is needed because, on entry into the kernel, the kernel's execution environment has not been set up, so it is not possible to access memory (or do much of anything). Executing `swapgs` is the first step toward establishing that environment, allowing access to kernel data and the kernel stack. Immediately prior to the return to user space, the kernel will execute another `swapgs` to restore the GS base register to its user-space value. 

If an attacker is able to redirect an indirect branch to land on `entry_SYSCALL_64()`, the hardware IBT check will pass, since the expected `endbr` instruction is present. The FineIBT hash check, though, will not happen, since that code is missing from the function preamble. As a result, hostile indirect calls to that function will be allowed to proceed. That is bad enough, but the `swapgs` instruction makes it far worse. It will restore the user-space GS-base value (the one that was replaced when the kernel was first entered) while the CPU is still running in kernel mode; user space is allowed to change that register, so the kernel's GS base will be set to a value that is entirely under the control of the attacker. Among other things, that puts the kernel stack under the attacker's control; the result is a quick takeover of the kernel. 

When spelled out in this way, the problem is reasonably obvious; checking a hash within the called function can only work if every function includes that checking — and there are functions, including `entry_SYSCALL_64()`, that cannot do the checking. Moving the checking to the caller avoids this problem, at the cost of making the entire sequence a bit more expensive. That is the approach that Zijlstra has taken; the code sequence that is used for this checking, found in [this patch](<https://lwn.net/ml/all/20250224124200.598033084@infradead.org>), merits a look: 

```
/*
         * Notably LEA does not modify flags and can be reordered with the CMP,
         * avoiding a dependency. Again, using a non-taken (backwards) branch
         * for the failure case, abusing LEA's immediate 0xf0 as LOCK prefix for the
         * Jcc.d8, causing #UD.
         */
        asm(	".pushsection .rodata			\n"
        	"fineibt_paranoid_start:			\n"
        	"	movl	$0x12345678, %r10d		\n"
        	"	cmpl	-9(%r11), %r10d			\n"
        	"	lea	-0x10(%r11), %r11		\n"
        	"	jne	fineibt_paranoid_start    0xd	\n"
        	"fineibt_paranoid_ind:				\n"
        	"	call	*%r11				\n"
        	"	nop					\n"
        	"fineibt_paranoid_end:				\n"
        	".popsection					\n"
        );
```

The `0x12345678` is patched at run time with the expected hash value. When the time comes to perform the indirect call, the `cmpl` instruction compares the patched-in value against the hash that is expected to be stored just prior to the entry point for the indirectly called function. The `lea` instruction is essentially a fast no-op, but it is there for the following cleverness. The `jne` instruction will look at the result of the `cmpl` two instructions before; in the not-equal case (the hash did not match) it jumps backward into the just-executed code; otherwise the `call` is executed as usual. 

Why the backward jump? Since branches that are not taken are faster than those that are, it is better to jump in the uncommon case. This particular jump is noteworthy, though, in that it will land _in the middle_ of the `lea` instruction, which will cause the CPU to see an invalid instruction sequence and generate a `#UD` trap. This trick, it seems, is the fastest and most space-efficient way that could be found to perform the test and generate the trap without slowing down legitimate indirect function calls (which should be all of them) any more than necessary. This special sequence is evidently the brainchild of Scott Constable at Intel; in [a previous version](<https://lwn.net/ml/all/20250219162107.880673196@infradead.org>) of the patch set, Zijlstra admonished: ""be warned, Scott loves overlapping instructions"". 

At the completion of this patch series, there are a couple of new options to the painstakingly undocumented `cfi=` command-line parameter. Setting `cfi=warn` causes control-flow-integrity errors to generate a warning rather than generating an oops, while `cfi=paranoid` enables the new verify-before-calling mode. Toward the end of the series, there is also [a patch](<https://lwn.net/ml/all/20250224124200.820402212@infradead.org>) adding another option, `cfi=bhi`, that improves the Spectre mitigations that are supposed to be built into IBT, but which have been found to be lacking at the hardware level in some processors. 

Zijlstra, in the cover letter, expressed the hope that the current version of the patch set would be the last before it is merged. Such hopes are often dashed in the kernel world, but this series would appear to be getting close to completion. It is not clear whether attackers have ever exploited the bypass reported by Miller but, once this code goes in, the authors of any such exploits will have to look for a new way to get around the kernel's control-flow-integrity protections.
