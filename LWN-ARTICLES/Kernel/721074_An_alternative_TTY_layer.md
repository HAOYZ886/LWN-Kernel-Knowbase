---
title: An alternative TTY layer
url: https://lwn.net/Articles/721074/
date: "April 27, 2017"
category: Embedded systems; TTY layer
author: "By Jonathan Corbet April 27, 2017"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
April 27, 2017

The Linux kernel is highly scalable but, while it runs nicely on the world's largest computers, it is not an entirely comfortable fit on the smallest. The difficulties involved in running Linux on machines with 1MB or less of memory have left an opening for other operating systems, such as [Zephyr](<https://www.zephyrproject.org/>), with lower memory needs. Some developers have not given up on scaling Linux to the smallest computers, but the approaches they have to take have always been a bit of a hard sell with the rest of the development community. Nicolas Pitre's [minitty](<https://lwn.net/Articles/718673/>) patch set is a case in point. 

The kernel's terminal (TTY) layer supports a long list of hardware and no end of features. It is a complex subsystem that must support decades of Unix and Linux history; core-kernel developer Ingo Molnar once [confessed](<https://lwn.net/Articles/243968/>) that the TTY code is ""one of the very few pieces of kernel code that scares the hell out of me"". It is also quite large, weighing in at about 46KB of compiled code without the low-level hardware drivers. 46KB is not a lot of memory on a typical desktop system, or even on a mobile handset but, in situations where every byte counts, it hurts. 

When Pitre set out to find a way to slim down the TTY subsystem for small systems, he ran into some problems. The TTY code has a multi-layer design to give it the maximum amount of flexibility; each layer adds overhead. Line disciplines — modules that manage the low-level protocols used on serial lines — can be plugged in and out, and the kernel supports about two dozen of them. The line disciplines themselves are easy to leave out, but the infrastructure that allows them to be modular is not optional. The TTY layer is designed to support pseudoterminals, and it is able to support dozens of ports running at full speed — an important feature in the early days of Linux. Pitre concluded that there is no way to make this infrastructure optional without worsening performance or functionality for those who need it. 

So, instead, he went and wrote an entirely new TTY subsystem called "minitty". About the only thing it can do is move data over a hardware serial port. There are no pseudoterminals, no choice of line disciplines, no job control, and not much of anything else. What it does provide is the ability to get debugging information out of the system via a serial port or to run simple login sessions. Minitty shares the TTY layer's low-level hardware drivers, though, and it provides a standard (if limited) interface to user space. And, importantly, it does that in 8KB of memory — about 1/6 of what is required by the standard TTY layer. 

Minitty seems like a clear win for extremely small Linux deployments, but it quickly ran into trouble on the mailing list. Kernel developers have a strong aversion to duplicated subsystems, and for a number of good reasons. This kind of duplication forces choices onto distributors and users that they may not be in a position to make. It can split development effort between multiple competing implementations, bugs present in both subsystems are often only fixed in one, and maintenance can be complicated if changes to one subsystem can break the other. So it is not entirely surprising that minitty has drawn some unfavorable responses. 

Greg Kroah-Hartman, the current maintainer of the TTY layer, [described](<https://lwn.net/Articles/721081/>) it as ""a horrid thing to do"" and advised Pitre to work with the existing code instead: 

So please, take what we have, refactor, and carve things up so that the _same_ code paths are being used for both "big and little" tty layers. That way _everyone_ benefits, no need to have totally separate code paths, and totally different files that different people maintain. 

Other developers also made their opposition to a separate TTY layer known. Pitre is not so easily discouraged, though. He [reiterated](<https://lwn.net/Articles/718401/>) that the TTY layer simply cannot be shrunk to the same degree: 

Let's be honest here: the existing code _could_ possibly be reduced of course. That would require a lot of efforts to gain 50% reduction maybe? What I'm looking at with my proposal here is a 6x reduction factor and I'm still not done with it. There is no way I could do that with the existing code. 

Most of his message (which is worth reading in its entirety) was devoted to explaining why he thinks a tiny TTY layer is so important. Linux is simply not a viable kernel for the smallest systems without the ability to make it as small as possible. But there is a great advantage to using Linux in such settings. Linux is well maintained and well debugged; security issues are fixed (relatively) quickly there. If a target system is running Linux, developers can write and debug their code on an ordinary desktop Linux system and, with a bit of care, expect it to work unmodified on the target system. All of the existing Linux-based development tools will be available, and existing Linux developers already know how to develop for such targets. 

The ability to shrink the kernel in this way is, according to Pitre, a crucial part of making Linux successful in the smallest Internet-of-things settings. A smaller TTY layer is only the beginning; he would eventually like to come up with miniaturized versions of many other kernel subsystems as well. Accommodating these mini-subsystems will require a bit of a change to the normal kernel development model, he said, but the rewards will be more than worth the cost. 

At least part of this message seems to have gotten through; Kroah-Hartman [responded](<https://lwn.net/Articles/721086/>) to the most recent posting by saying: ""I agree with your goal of getting Linux running on these very tiny chips, I want that to happen too"". If minitty can find its way to acceptance, that may eventually open the door to other tinification efforts; one should remember that this is the kernel community we're talking about, though, so expect a lengthy discussion for each proposal. Getting this kind of work merged will never be easy but, just maybe, the community will be able to accept enough of it to help ensure that the kernel is competitive in the smallest of deployments.
