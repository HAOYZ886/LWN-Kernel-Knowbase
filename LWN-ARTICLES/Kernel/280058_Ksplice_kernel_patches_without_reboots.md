---
title: "Ksplice: kernel patches without reboots"
url: https://lwn.net/Articles/280058/
date: "April 29, 2008"
category: "Ksplice; Live patching; Security-Security technologies"
author: "By Jonathan Corbet April 29, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 29, 2008

The kernel developers are generally quite good about responding to security problems. Once a vulnerability in the kernel has been found, a patch comes out in short order; system administrators can then apply the patch (or get a patched kernel from their distributor), reboot the system, and get on with life knowing that the vulnerability has been fixed. It is a system which works pretty well. 

One little problem remains, though: rebooting the system is a pain. At a minimum, it requires a few minutes of down time. In many situations, that down time cannot be tolerated. Reboots also disrupt any ongoing work, break existing network connections, and can cause the loss of results from long-running processes. And, most importantly of all, reboots prove traumatic for a certain subset of Linux administrators who prize a long uptime above almost all other things. Administrators currently have to choose between multi-year uptimes and security fixes; anything which frees them from a dilemma of this magnitude can only be welcome. 

That "anything" might just be a recently-[announced](<http://lwn.net/Articles/279378/>) project called [ksplice](<http://web.mit.edu/ksplice/>). With ksplice, system administrators can have the best of both worlds: security fixes without unsightly reboots. 

An in-depth explanation of how ksplice works can be found in [this document [PDF]](<http://web.mit.edu/ksplice/doc/ksplice.pdf>). In short, ksplice requires as input the source tree for the running kernel and the security patch. It will then build two kernels, one with the patch and one without; the kernels are built with a special set of options which makes it easy to figure out which functions change as a result of the patch. The two kernels will be compared, with the purpose of finding those functions. Changes can propagate further than one might expect, especially if, for example, an inline function is modified. 

Once a list of changed functions has been made, the updated code for those functions is packaged into a kernel module and loaded into the system. Then comes the tricky part: getting the running kernel to start using the new code. That requires patching the running code, which is a risky thing to do. Ksplice starts with a call to `stop_machine_run()`, which dumps a high-priority thread onto each processor, thus taking control of all processors in the system. It then examines all threads in the system to ensure that none of them are running in the functions to be replaced; if so, trampoline jumps are patched into the beginning of each replaced function (they "bounce" the call to the old code into the replacement code) and life continues. Otherwise ksplice will back off and try again later. 

This method imposes a number of limitations. One is that only code changes can be patched in with ksplice; patches which make changes to data structures cannot be accommodated. Another comes from the retry-based approach to ensuring that no threads are running in the patched functions; what happens if one of those functions is never free? Kernel functions like `schedule()`, `sys_poll()`, or `sys_waitid()` are likely to always have processes running within them. In cases like this, ksplice will eventually give up and inform the user that the patch cannot be done; it is simply not possible to make changes to those particular functions. 

These limitations mean that, out of 50 security patches examined by the ksplice developers, eight could not be applied with ksplice. So multi-year uptimes are probably still incompatible with the application of all security patches. Even so, ksplice certainly has the potential to reduce patch-related downtime considerably. Chances are good that there will be a fair amount of interest in ksplice in sites running high-uptime, mission-critical systems. 

There are few things in the way of an immediate merge of this code into the mainline. One is a matter of coding quality and can be fixed. Then, there is the matter of the lead developer [being unconvinced](<https://lwn.net/Articles/280064/>) that merging this code makes sense since it is, essentially, a standalone feature. Andi Kleen's [response](<https://lwn.net/Articles/280065/>) made the (usual) reasons for merging the code clear: 

To be honest you weren't the first to come up with something like this (although you're the first to post to l-k as far as I know). But the usual problem of something that is kept out of tree is that it eventually bitrots and gets forgotten. The only sane way to make such extensions a generically usable linux feature is to merge them to mainline. 

So, presumably, the code will eventually be proposed for a mainline merge. But there is one other little difficulty [pointed out](<https://lwn.net/Articles/280066/>) by Tomasz Chmielewski: Microsoft holds [a patent](<http://www.google.com/patents?id=cVyWAAAAEBAJ&dq=hotpatching>) described this way: 

A system and method for automatically updating software components on a running computer system without requiring any interruption of service. A software module is hotpatched by loading a patch into memory and modifying an instruction in the original module to jump to the patch. 

Microsoft came up with this novel new technique in the distant past: 2002. The posting immediately brought out a crowd of surprised graybeards who distinctly remember using such techniques on their PDP-11 systems some decades before Microsoft "invented" hot-patching. The basic claim of the patent would thus appear to be invalidated by some decades' worth of prior art, but some of the dependent claims include features (such as capturing all other processors on the system) which were unlikely to be useful on PDP-11s. 

Given that the kernel developers are now well aware of this patent, they must take it into account when deciding whether to accept this code into the mainline. It would not be surprising if they chose to avoid baiting the Microsoft FUD machine in this way, even if they all agreed that the patent lacked validity. So a promising technology risks being left out of the kernel as the result of a software patent which was filed at least 30 years too late.
