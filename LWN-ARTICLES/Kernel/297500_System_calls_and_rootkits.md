---
title: System calls and rootkits
url: https://lwn.net/Articles/297500/
date: "September 10, 2008"
category: Security; System calls
author: "By Jake Edge September 10, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
September 10, 2008

A patch to add some security checks before making system calls would seem like a reasonable addition to the kernel, but because it is, at best, a half-measure, it received a less than enthusiastic response. Preventing rootkits--malware that alters the kernel to hide its presence and function--from altering the system call table was the rationale behind the patch, but it would only work for the current crop of rootkits. Once that change was made, rootkit authors would just change their _modus operandi_ in response. 

There are many possible ways that a root user--or malware running as root--can modify a Linux system to run rootkit code. Some currently "popular" rootkits modify the system call table, though it is ostensibly read-only. Some commercial malware scanners that run on Linux have also been known to use this technique. In both cases, certain system calls are re-routed from the standard kernel code to code that lives elsewhere. That code, running in kernel mode, can then do just about anything it wants with the system. 

Arjan van de Ven proposed a [patch](<http://lwn.net/Articles/296926/>) that hooked into the system call entry code to check the address of the call to ensure that it was within the addresses occupied by kernel code. He describes the change and its impact this way: 

The patch below, while obviously not perfect protection against malware, adds some cheap sanity checks to the syscall path to verify the system call is actually still in the kernel code region and not some external-to-this region such as a rootkit. 

The overhead is very minimal; measured at 2 cycles or less. (this is because the branches get predicted right and the rest of the code is almost perfectly parallelizable... and an indirect function call is a branch issue anyway) 

Various kernel hackers pointed out the flaws inherent in that scheme. As Andi Kleen succinctly [puts it](<https://lwn.net/Articles/297628/>): 

This just means that the root kits will switch to patch the first instruction of the entry points instead. [...] So the protection will be zero to minimal, but the overhead will be there forever. 

One of the more interesting ideas to come out of the discussion was Alan Cox's [thoughts](<http://lwn.net/Articles/297622/>) on using a hypervisor to enforce protections: 

The only place you can expect to make a difference here is in virtualised environments by teaching KVM how to provide 'irrevocably read only' pages to guests where the guest OS isn't permitted to change the rights back or the virtual mapping of that page. 

Ingo Molnar [described](<https://lwn.net/Articles/297631/>) a rather complicated scheme that might increase the likelihood of a rootkit being detected, but with a fairly high cost--in build complexity as well as the ability to debug the resulting kernel. The compiler would be changed to insert calls to rootkit checks randomly throughout the kernel binary in ways that would be difficult or impossible for a rootkit to detect and evade. In the end, though, a rootkit could simply install a new kernel that does exactly what it wants, then cause, or wait for, a reboot. 

Without some kind of hardware enforcement (e.g. [Trusted Platform Module](<http://en.wikipedia.org/wiki/Trusted_Platform_Module>)) or locked-down virtualization, Linux is defenseless against attacks that run as root. The kernel could change to thwart a particular kind of attack, such as van de Ven's patch, but other kinds of attacks will still succeed. It is clearly a situation where ""the only way to win is not to play this game"", as Pavel Machek--amongst others--[noted](<https://lwn.net/Articles/297666/>) in the thread. 

In the end, van de Ven wrote off the patch as an exercise in measuring the cost of this kind of runtime checking. It was fairly low cost solution, but without any major upside. The real upside was getting kernel hackers thinking about the problem, which could lead to some better solutions down the road.
