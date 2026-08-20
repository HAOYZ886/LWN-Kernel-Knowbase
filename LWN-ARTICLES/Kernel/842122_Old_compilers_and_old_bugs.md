---
title: Old compilers and old bugs
url: https://lwn.net/Articles/842122/
date: "January 11, 2021"
category: Build system
author: "By Jonathan Corbet January 11, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 11, 2021

The kernel project goes out of its way to facilitate building with older toolchains. Building a kernel on a new system can be enough of a challenge as it is; being forced to install a custom toolchain first would not improve the situation. So the kernel developers try to keep it possible to build the kernel with the toolchains shipped by most distributors. There are costs to this policy though, including an inability to use newer compiler features. But, as was seen in a recent episode, building with old compilers can subject developers to old compiler bugs too. 

On January 5, Russell King [reported](<https://lwn.net/ml/linux-kernel/20210105154726.GD1551@shell.armlinux.org.uk/>) on a problem he had been chasing for a long time. Some of his 64-bit Arm systems running 5.4 or later kernels would, on rare occasion, report a checksum failure on the ext4 root filesystem. It could take up to three months of uptime for the problem to manifest itself, making it, as King described it, ""unrealistic to bisect"". He had, however, found a way to more reliably reproduce the failure, making the task of finding out when the problem was introduced plausible, at least. 

Starting with King's findings, a number of developers working in the Arm subsystem looked into the issue; their efforts seemed to point out [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=22ec71615d82>) as the culprit. That change, applied in 2019, relaxed the memory barriers used around I/O accessors, optimizing accesses to I/O memory. Reverting this patch made the problem go away. 

Some developers might have applied the revert and called the problem solved, but that is not what happened here. Will Deacon, the author of the patch in question, was convinced of its correctness; if the Arm architecture is behaving as specified, there should be no need for the stronger barriers, so something else was going on. Reverting the patch, in other words, made the issue go away by papering over a real problem somewhere else. 

Where might that "somewhere else" be? King suggested that it could be somewhere else in the kernel, in the Arm processor itself, or in the cache-coherent interconnect that ties together processor clusters and memory. He thought that a problem in the hardware was relatively unlikely, and that the bug thus lurked somewhere within the kernel. That, naturally, led to a lot of code examination, especially within the ext4 filesystem. 

Two days later, King [announced](<https://lwn.net/ml/linux-kernel/20210107111841.GN1551@shell.armlinux.org.uk/>) that the problem had been found; it indeed was an issue within the ext4 filesystem, but not of the variety that had been expected. A look at the assembly code generated for `ext4_chksum()` revealed that the compiler was freeing the function's stack frame prior to the end of the function itself. The last line of the function is: 

```
return *(u32 *)desc.ctx;
```

Here, `desc` is a local variable, living on the stack. The compiled function was resetting the stack pointer above this variable immediately before fetching `desc.ctx`. That led to a window of exactly one instruction where the function was using stack space that had already been freed. 

This is a compiler bug of the worst type. The miscompiled code will work as expected almost every time; there is, after all, no other code trying to allocate stack space in that one-instruction window. All bets are off, though, if an interrupt arrives exactly between the two instructions; then the stack will be overwritten and the load of `desc.ctx` will be corrupted, leading to the observed checksum failure. This is something that will _almost_ never happen, but when it does things will go badly wrong. 

This miscompilation was done by GCC 4.9.4, which was released in August 2016 (4.9.0, the major release on which it is based, came out in April 2014). The relevant bug, though, was [reported](<https://gcc.gnu.org/bugzilla/show_bug.cgi?id=63293>) in 2014 and fixed in November of that year. That fix was seemingly never backported from the (then) under-development 5.x release to 4.9.x, so the 4.9.4 release did not contain it. Interestingly, versions of 4.9.4 shipped by distributors like Red Hat, Android, and Linaro all _did_ have the fix backported, so it only affected developers not using those versions. The bug lurked there for years until finally turning up in King's builds. 

One outcome from this episode is a clear illustration of the potential downside of supporting old toolchains. A great deal of effort went into tracking down a bug that had, in fact, been fixed six years ago; that would have not been necessary if developers were not still using 4.9.x compilers. 

As it happens, GCC 4.9 is the oldest compiler supported by the kernel, but even that requirement is relatively recent. [As of 2018](<https://lwn.net/Articles/748074/>), the kernel still claimed (not entirely truthfully) that it could be built with GCC 3.2, which was released in 2002. As a result of discussions held in 2018, the minimum GCC version was moved forward to 4.6; later it became 4.9. 

Fixing GCC 4.9 to address this bug is out of the question; the GCC developers have long since moved on from that release. So, at a minimum, the oldest version of the compiler that can be used for the arm64 architecture will have to be moved forward to 5.1. But that immediately led to the question of whether the oldest version for _all_ architectures should be moved forward. 

Ted Ts'o was [in favor](<https://lwn.net/ml/linux-kernel/X/c2aqSvYCaB9sR6@mit.edu/>) of that change, but he also pointed out that RHEL 7 (and thus CentOS 7) systems are still stuck with GCC 4.8. As Peter Zijlstra [noted](<https://lwn.net/ml/linux-kernel/X/giM+tM3FGVj0Np@hirez.programming.kicks-ass.net/>), though, it is already necessary to install a newer compiler than the distribution provides to build the kernel on those systems. Arnd Bergmann [said](<https://lwn.net/ml/linux-kernel/CAK8P3a2svyz1KXSqSUMVeDqdag4f1VcERH9jpECSLsn-FWvZbw@mail.gmail.com/>) that the other known users of GCC 4.9 were Android and Debian 8\. Android has since switched over to Clang to build its kernels, and Debian 8 [went unsupported](<https://www.debian.org/News/2020/20200709>) at the end of June 2020. So it would appear that relatively few users would be inconvenienced by raising the minimum GCC version to 5.1. 

On the other hand, there are some advantages to such a move beyond leaving an unpleasant bug behind. Bergmann [argued](<https://lwn.net/ml/linux-kernel/CAK8P3a2J8fLjPhyV0XUeuRBdSo6rz1gU4wrQRyfzKQvwhf22ag@mail.gmail.com/>) for this change because it would allow compiling the kernel with `-std=gnu11`, making it possible to rely on bleeding-edge [C11](<https://en.wikipedia.org/wiki/ANSI_C#C11>) features. Currently, kernel builds use `-std=gnu89`, based on the rather less shiny [C89](<https://en.wikipedia.org/wiki/ANSI_C#C89>) standard. [Zijlstra](<https://lwn.net/ml/linux-kernel/X/gkMmObbkI4+ip/@hirez.programming.kicks-ass.net/>) and [Deacon](<https://lwn.net/ml/linux-kernel/20210108092655.GA4031@willie-the-truck/>) both added that moving to 5.1 would allow the removal of a number of workarounds for GCC 4.9 problems. 

Given all that, it seems unlikely that there will be much opposition to moving the kernel as a whole to the 5.1 minimum version. That said, Linus Torvalds [is unconvinced](<https://lwn.net/ml/linux-kernel/CAHk-=whnKkj5CSbj-uG_MVVUsPZ6ppd_MFhZf_kpXDkh2MAVRA@mail.gmail.com/>) about the value of such a change and may yet need some convincing. Even if the shift to 5.1 does not happen right away, the writing would seem to be on the wall that GCC 4.9 will not be supported indefinitely. GCC 5.1, released in April 2015, is not the newest thing on the planet either, of course. But hopefully it has fewer lurking bugs while simultaneously making some welcome new features available. Supporting old toolchains has its value, but so does occasionally dropping the oldest of them.
