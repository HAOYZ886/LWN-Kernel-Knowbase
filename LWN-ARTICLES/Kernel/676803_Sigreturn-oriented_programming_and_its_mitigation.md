---
title: "Sigreturn-oriented programming and its mitigation"
url: https://lwn.net/Articles/676803/
date: "February 24, 2016"
category: Security
author: "By Jonathan Corbet February 24, 2016"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
February 24, 2016

In the good old days (from one point of view, at least), attackers had an easy life; all they had to do was to locate a buffer overrun vulnerability, then they could inject whatever code they liked into the vulnerable process. Over the years, kernel developers have worked to ensure that data that can be written by an application cannot be executed by that application; that has made simple code-injection unfeasible in most settings. Attackers have responded with techniques like [return-oriented programming](<https://en.wikipedia.org/wiki/Return-oriented_programming>) (ROP), but ROP attacks are relatively hard to get right. On some systems, attackers may be able to use the simpler sigreturn-oriented programming (SROP) technique instead; kernel patches have been circulating in an attempt to head off that class of attacks. 

#### Some background

If data on the stack cannot be executed, a buffer overflow vulnerability cannot be used to inject code directly into an application. Such vulnerabilities _can_ , however, be used to change the program counter by overwriting the current function's return address. If the attacker can identify code existing within the target process's address space that performs the desired task, they can use a buffer overflow to "return" to that code and gain control. 

Unfortunately for attackers, most programs lack a convenient "give me a shell" location to jump to via an overwritten return address. But it is still likely that the program contains the desired functionality; it is just split into little pieces and scattered throughout the address space. The core idea behind return-oriented programming is to find these pieces in places where they are followed by a return instruction. The attacker, who controls the stack, can not only jump to the first of these pieces; they can also place a return address on the stack so that when this piece executes its return instruction, control goes to another attacker-chosen location — the next piece of useful code. By stringing together a set of these "gadgets," the attacker can create a new program within the target process. 

There are various tools out there to help with the creation of ROP attacks. Scanners can pass through an executable image and identify gadgets of interest. "ROP compilers" can then create a program to accomplish the attacker's objective. But the necessary gadgets may not be available, and techniques like address-space layout randomization (ASLR) make ROP attacks harder. So ROP attacks tend to be fiddly affairs, often specific to the system being attacked (or even to the specific running process). Attackers, being busy people like the rest of us, cannot be blamed if they look for easier ways to compromise a system. 

#### Exploiting sigreturn()

Enter `sigreturn()`, a Linux system call that nobody calls directly. When a signal is delivered to a process, execution jumps to the designated signal handler; when the handler is done, control returns to the location where execution was interrupted. Signals are a form of software interrupt, and all of the usual interrupt-like accounting must be dealt with. In particular, before the kernel can deliver a signal, it must make a note of the current execution context, including the values stored in all of the processor registers. 

It would be possible to store this information in the kernel itself, but that might make it possible for an attacker (of a different variety) to cause the kernel to allocate arbitrary amounts of memory. So, instead, the kernel stores this information on the stack of the process that is the recipient of the signal. Prior to invoking the signal handler, the kernel pushes an (architecture-specific) variant of the [`sigcontext`](<http://lxr.free-electrons.com/source/arch/x86/include/uapi/asm/sigcontext.h#L237>) structure onto the process's stack; this structure contains register information, floating-point status, and more. When the signal handler has completed its job, it calls `sigreturn()`, which restores all that information from the on-stack structure. 

Attackers employing ROP techniques have to work to find gadgets that will store the desired values into specific processor registers. If they can call `sigreturn()`, though, life gets easier, since that system call sets the values of all registers directly from the stack. As it happens, the kernel has no way to know whether a specific `sigreturn()` call comes from the termination of a legitimate signal handler or not; the whole system was designed so that the kernel would not have to track that information. So, as Erik Bosman and Herbert Bos noted in [this paper [PDF]](<http://www.cs.vu.nl/~herbertb/papers/srop_sp14.pdf>), `sigreturn()` looks like it might be helpful to attackers. 

There is one obstacle that must be overcome first, though: an attacker must find a ROP gadget that makes a call to `sigreturn()` — and few applications do that directly. One way to do that would be to locate a more generic gadget for invoking system calls, then arrange for the appropriate number to be passed to indicate `sigreturn()`. But in many cases that is unnecessary; for years, the kernel developers conveniently put a `sigreturn()` call in a place where attackers could find it — at a fixed address that is not subject to ASLR. That address is in the "virtual dynamic shared object" (vDSO) area, a page mapped by the kernel in a known location into every process to optimize some system calls. On other systems, the `sigreturn()` call can be found in the C library; exploiting that one requires finding a way to leak some ASLR information first. 

Bosman and Bos demonstrated that `sigreturn()` can be used to exploit processes with a buffer overflow vulnerability. Often, the `sigreturn()` gadget is the only one that is required to make the exploit work; in some cases, the exploit can be written in a system-independent way, able to be reused with no additional effort. More recent kernels have made these exploits harder (the vDSO area is no longer usable, for example), but they are still far from impossible. And, in any case, many interesting targets are running older kernels. 

#### Stopping SROP

Scott Bauer recently posted [a patch set](<https://lwn.net/Articles/674861/>) meant to put an end to SROP attacks. Once the problem is understood, the solution becomes clear relatively quickly: the kernel needs a way to verify that a `sigcontext` structure on the stack was put there by the kernel itself. That would ensure that `sigreturn()` can only be called at the end of a real signal delivery. 

Scott's patch works by generating a random "cookie" value for each process. As part of the signal-delivery process, that cookie is stored onto the stack, next to the `sigcontext` structure. Prior to being stored, it is XORed with the address of the stack location where it is to be stored, making it a bit harder to read back; future plans call for hashing the value as well, making the recovery of the cookie value impossible. Even without hashing, though, the cookie should be secure enough; an attacker who can force a signal and read the cookie off the stack is probably already in control. 

The `sigreturn()` implementation just needs to verify that the cookie exists in the expected location; if it's there, then the call is legitimate and the call can proceed. Otherwise the operation ends and a `SIGSEGV` signal is delivered to the process, killing it unless the process has made other arrangements. 

There are some practical problems with the patch still; for example, it will not do the right thing in settings where [checkpoint-restore in user space](<http://criu.org/Main_Page>) is in use (a restored process will have a new and different random cookie value, but old cookies may still be on the stack). Such problems can be worked around, but they may force the addition of a sysctl knob to turn this protection off in settings where it breaks things. It also does nothing to protect against ROP attacks in general, it just closes off one relatively easy-to-exploit form of those attacks. But, as low-hanging fruit, it is probably worth pursuing; there is no point in making an attacker's life easier.
