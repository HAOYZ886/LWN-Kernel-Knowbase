---
title: Splicing out syscalls for tiny kernels
url: https://lwn.net/Articles/624258/
date: "December 3, 2014"
category: Embedded systems
author: "By Jonathan Corbet December 3, 2014"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 3, 2014

It is no secret that the Linux kernel has grown over time; the constant addition of features and hardware support means that almost every development cycle adds more code than it removes. The good news is that, for most of us, the increase in hardware speed and size has far outstripped the growth of the kernel, so few of us begrudge the extra resources that a larger kernel requires. Developers working on tiny systems, though, are still concerned about every byte consumed by the kernel. Accommodating their needs seems likely to be a source of ongoing stress in the community. 

The latest example comes from [Pieter Smith's patch set](<https://lwn.net/Articles/624259/>) to remove support for the `splice()` family of system calls, including `sendfile()` and `tee()`. There will be many tiny systems with dedicated applications that have no need for those calls; removing them from the kernel makes 8KB of memory available for other purposes. The Linux "tinification" developers see that as a worthwhile gain, but some others disagree. 

In particular, David Miller [opposed the change](<https://lwn.net/Articles/624260/>), saying ""I think starting to compile out system calls is a very slippery slope we should not begin the journey down."" He worries that, even if a specific system works today without `splice()`, there may be a surprise tomorrow when some library starts using that system call. Developers working on Linux systems, David appears to be arguing, should be able to count on having the basic system call set available to them anywhere. 

The tinification developers have a couple of answers to this concern. One is that developers working on tiny systems know what they are doing and which system calls they can do without. As Josh Triplett [put it](<https://lwn.net/Articles/624261/>): 

We're talking about embedded systems small enough that you're booting with init=/your/app and don't even call fork(), where you know exactly what code you're putting in and what libraries you use. And they're almost certainly not running glibc. 

The other response is that the kernel has, in fact, provided support for compiling out major subsystems since the beginning. [Quoting Josh](<https://lwn.net/Articles/624262/>) again: 

It's not a "slippery slope"; it's been our standard practice for ages. We started down that road long, long ago, when we first introduced Kconfig and optional/modular features. /dev/* are user-facing interfaces, yet you can compile them out or make them modular. /sys/* and/proc/* are user-facing interfaces, yet you can compile part or all of them out. Filesystem names passed to mount are user-facing interfaces, yet you can compile them out. 

(This list goes on for some time; see the original mail for all the details). Eric Biederman [added](<https://lwn.net/Articles/624263/>) that the SYSV IPC system calls have been optional for a long time, and Alan Cox [listed more optional items](<https://lwn.net/Articles/624264/>) as well. David finally [seemed to concede](<https://lwn.net/Articles/624265/>) that making system calls optional was not a new thing for the Linux kernel, but he stopped short of actually supporting the `splice()` removal patch. 

Without his opposition, though, this patch may go in. But a look at [the kernel tinification project list](<https://tiny.wiki.kernel.org/projects>) makes it clear that this discussion is likely to return in the future. The tinification developers would like to be able to compile out support for SMP systems, random number generation, signal handling, capabilities, non-root users, sockets, the ability for processes to exit, and more. Eventually, they would like to have an automated tool that can examine a user-space image and build a configuration removing every system call that the given programs do not use. 

Needless to say, any kernel that has been stripped down to that extent will not resemble a contemporary Linux system. But, on the other hand, neither do the ancient (but much smaller) kernels that these users often employ now. If Linux wants to have a place on tiny systems, the kernel will have to adapt to the resource constraints that come with such systems. That will bring challenges beyond convincing developers to allow important functionality to be configured out; the tinification developers will also have to figure out a way to allow this configuration without introducing large numbers of new configuration options and adding complexity to the build system. 

It looks like a hard line to walk. But the Linux kernel embodies the solution to a lot of hard problems already; where there are willing developers, there is usually a way. If the tinification developers can find a way here, Linux has a much better chance of being present on the tiny systems that are likely to be embedded in all kinds of devices in the coming years. That seems like a goal worth trying for.
