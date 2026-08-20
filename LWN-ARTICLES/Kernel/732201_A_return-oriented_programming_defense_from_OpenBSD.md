---
title: "A return-oriented programming defense from OpenBSD"
url: https://lwn.net/Articles/732201/
date: "August 30, 2017"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet August 30, 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 30, 2017

Stack-smashing attacks have a long history; they featured, for example, as a core part of the [Morris worm](<https://en.wikipedia.org/wiki/Morris_worm>) back in 1988\. Restrictions on executing code on the stack have, to a great extent, put an end to such simple attacks, but that does not mean that stack-smashing attacks are no longer a threat. [Return-oriented programming (ROP)](<https://en.wikipedia.org/wiki/Return-oriented_programming>) has become a common technique for compromising systems via a stack-smashing vulnerability. There are various schemes out there for defeating ROP attacks, but a mechanism called "RETGUARD" that is being implemented in OpenBSD is notable for its relative simplicity. 

In a classic stack-smashing attack, the attack code would be written directly to the stack and executed there. Most modern systems do not allow execution of on-stack code, though, so this kind of attack will be ineffective. The stack _does_ affect code execution, though, in that the call chain is stored there; when a function executes a "return" instruction, the address to return to is taken from the stack. An attacker who can overwrite the stack can, thus, force a function to "return" to an arbitrary location. 

That alone can be enough to carry out some types of attacks, but ROP adds another level of sophistication. A search through a body of binary code will turn up a great many short sequences of instructions ending in a return instruction. These sequences are termed "gadgets"; a large program contains enough gadgets to carry out almost any desired task — if they can be strung together into a chain. ROP works by locating these gadgets, then building a series of stack frames so that each gadget "returns" to the next. 

This technique allows the construction of arbitrary programs on the stack without the need for execute permission on the stack itself. It is worth noting that, on a complex-instruction-set architecture like x86, unexpected gadgets can be created by jumping into the middle of a multi-byte instruction, a phenomenon termed "polymorphism". Needless to say, there are tools out there that can be used by an attacker to locate gadgets and string them together into programs. 

The [RETGUARD mechanism](<https://lwn.net/Articles/732202/>), posted by Theo de Raadt on August 19, makes use of a simple return-address transformation to disrupt ROP chains and prevent them from executing as intended. It takes the form of a patch to the LLVM compiler adding a new `-fret-protector` flag. When code is compiled with that flag, two things happen: 

  * The prologue to each function (the code that runs before the body of the function itself) exclusive-ORs the return address on the stack with the value of the stack pointer itself. 

  * The epilogue, run just before the function returns, repeats the operation to restore the return address to its initial value. 

The exclusive-OR operation changes the return address into something that is effectively random, especially when address-space layout randomization is used to place the stack at an unpredictable location. With this change, the first gadget used by a ROP sequence will, when it attempts the second step above, transform the return address into something unpredictable and, most likely, useless to an attacker. That will stop the chain and thwart the attack. 

There is, of course, a significant limitation here: a ROP chain made up of exclusively polymorphic gadgets will still work, since those gadgets were not (intentionally) created by the compiler and do not contain the return-address-mangling code. De Raadt acknowledged this limitation, but said: ""we believe once standard-RET is solved those concerns become easier to address separately in the future. In any case a substantial reduction of gadgets is powerful"". 

Using the compiler to insert the hardening code greatly eases the task of applying RETGUARD to both the OpenBSD kernel and its user-space code. At least, that is true for code written in a high-level language. Any code written in assembly must be changed by hand, though, which is a fair amount of work. De Raadt and company have done that work; he reports that: ""We are at the point where userland and base are fully working without regressions, and the remaining impacts are in a few larger ports which directly access the return address (for a variety of reasons)"". It can be expected that, once these final issues are dealt with, OpenBSD will ship with this hardening enabled. 

It makes sense to ask whether this relatively straightforward hardening technique could be applied to the Linux kernel as well. Using LLVM to build the kernel is not yet a viable option, but it should be possible to reimplement the RETGUARD transformations as a GCC plugin module. The tiresome task of fixing up the assembly code would also need to be done; the [`objtool` utility](<https://lwn.net/Articles/658333/>) could probably be pressed into service to help with this task. But the patch that emerged would not be small. 

If any benchmarks have been run to determine the cost of using RETGUARD, they have not been publicly posted. The extra code will make the kernel a little bigger, and the extra overhead on every function is likely to add up in the end. But if this technique can make the kernel that much harder to exploit, it may well justify the extra execution overhead that it brings with it. All that's needed is somebody to actually do the work and try it out.
