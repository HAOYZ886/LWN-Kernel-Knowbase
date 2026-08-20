---
title: Running the kernel in library mode
url: https://lwn.net/Articles/639333/
date: "April 8, 2015"
category: "Library mode; Virtualization-Library mode"
author: "By Jonathan Corbet April 8, 2015"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 8, 2015

Once upon a time, the only way to run the Linux kernel was as the primary operating system on a handy piece of hardware. Since then, though, other modes of operation have become possible: the kernel can, for example, be run as the guest of another kernel through virtualization, or as a user-space process with the [user-mode Linux (UML)](<http://user-mode-linux.sourceforge.net/>) port. One mode that has not been supported is running the kernel as a library that can be called from within an application program, but that situation appears to be about to change thanks to a patch set which has just made its first appearance on the linux-kernel list. 

This patch set, posted by Hajime Tazaki, goes by the name [LibOS](<https://lwn.net/Articles/637658/>); it was presented ([slides [slideshare]](<http://www.slideshare.net/hajimetazaki/library-operating-system-for-linux-netdev01>)) at the recent [Netdev 0.1](<https://netdev01.org/>) conference. LibOS is structured as if it were a new architecture port; it can be found under `arch/lib` in the kernel tree. But this port, when built, does not result in a bootable kernel; instead, it creates a shared library that can then be loaded into a running process. 

One might wonder why this mode of operation would be useful. Though it is not limited to this particular use, the main focus of LibOS at the moment is to make the Linux network stack available to user-space applications. User-space network stacks are not unheard of in the Linux world; they have shown up in certain performance-sensitive settings for some years now. With LibOS, it is not necessary to write (or port) a new network stack to run in a Linux process; the kernel's network stack is now available to use directly. 

Needless to say, one does not just make the network stack callable from user space without doing a bit of work. To make this mode possible, the LibOS developers have created a whole set of stub functions to replace various kernel functions used by the networking code. Indeed, the bulk of the patch set consists of thousands of lines of stub functions. They do things like replacing the slab allocator with a simple version based on `malloc()` and, for the most part, shorting out the filesystem layer entirely. When that is done, what's left is the networking stack with almost enough scaffolding to let it run standalone within a process's address space. 

"Almost enough" because a few tasks are still left to the calling application. For example, there is no stub implementation of `schedule()`; instead, the calling code must provide one during the initialization process. The idea here is that the running application may want to exert some control over how the management of processes (most likely implemented as POSIX threads) will be done. 

There are currently two projects using the LibOS framework. [Networking in user space](<https://github.com/libos-nuse/linux-libos-tools>) (NUSE) finishes the job of providing a running user-space network stack. With NUSE, one can set up arbitrary networking topologies, interface to other user-space mechanisms like [DPDK](<http://dpdk.org/>) for fast transmission and reception of packets, and more. The [NS-3 system](<https://www.nsnam.org/overview/what-is-ns-3/>), instead, is a simulation framework used to run tests on network protocols and implementations. It can run network-oriented applications on top of the LibOS network stack using `LD_PRELOAD` tricks to redirect calls to the networking system calls. 

There are a number of interesting things that can be done with these tools. Users running networking in user space for performance reasons could consider using it, though the kernel's stack has not been optimized for performance in that setting. Somebody wanting to run an experimental protocol like [MPTCP](<https://lwn.net/Articles/544399/>) in production could use LibOS (built with a suitably patched kernel) to get that feature without touching the network stack used by the rest of the system. There are also a lot of opportunities for running debugging tools with a network stack that is running in user mode. 

While the LibOS work has been focused on the network stack as the first objective, there is nothing in its design that limits it to networking. If one wanted to, say, isolate the virtual filesystem layer instead, it would mostly be a matter of coming up with the additional stub functions needed. 

A question that might come to mind is: how does this differ from the user-mode Linux port that has been in the kernel for many years? Indeed, UML maintainer Richard Weinberger [wondered](<https://lwn.net/Articles/639347/>) exactly that. There appear to be a few differences. UML is meant to run as a standalone application in its own right, while LibOS runs as a library called by some other application. One can even have several LibOS instances running simultaneously within the same application. Beyond that, the idea of isolating a single subsystem for use within an application is not a part of the design of UML. After looking more deeply at the LibOS code, Richard [agreed](<https://lwn.net/Articles/639348/>) that it brought some interesting things to the table. 

One possible area of concern is the maintenance of all of the stub functions. There are a lot of them, and they will need to be updated whenever the corresponding "real" version is changed in the kernel. Few maintainers are likely to think that they have to update LibOS when they are making changes to their own subsystems. As a result, it seems likely that LibOS will be broken much of the time. 

That, in turn, means that maintenance concerns may be one of the chief obstacles LibOS must overcome before it can be considered for merging into the mainline kernel. If LibOS is often broken, developers will hesitate to use it. If LibOS breakage leads to complaints against subsystem maintainers working on their own code, they may respond by calling for its removal. Avoiding these pitfalls may require finding some way to automate the creation of these stub functions. Creating a library-mode version of the kernel may turn out to have been the easy part when one considers what is required to make that work maintainable in the long run.
