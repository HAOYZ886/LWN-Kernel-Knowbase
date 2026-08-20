---
title: Crash dumps with kexec
url: https://lwn.net/Articles/108595/
date: "October 27, 2004"
category: Crash dumps; Debugging; Kexec
author: 
---

One of the longstanding wishlist items for the Linux kernel is a built-in crash dump capability. Companies providing support for Linux, such as vendors of "enterprise" distributions, want this capability for the help it can provide in tracking down those obnoxious problems which only happen at the customer's site. Numerous implementations exist, but none have made it into the mainline kernel. Among the reasons for this is a lack of comfort with the crash dump code itself. That code runs when the state of the system is known to be compromised; people tend to worry that the kernel, in that state, could do unpleasant things, like corrupting filesystems. Even code which takes pains to never touch a disk is not entirely to be trusted when the system is reeling from a panic. 

The -mm tree contains an approach to crash dumps which may inspire a bit more trust. The core idea is to get the failing kernel out of the way entirely, as soon as possible, and to boot into a new kernel which can handle the real crash dump tasks. 

The mechanism used is the [kexec](<https://lwn.net/Articles/15468/>) system call, which loads and boots directly into a new kernel. The original goal was simply to speed up reboots by avoiding the BIOS and the whole set of time-consuming boot-time rituals which it performs; it's the sort of feature which appeals to kernel developers. Kexec patches have been circulating for some time, though the call has yet to make its way into a mainline kernel. 

Using kexec to perform crash dumps requires some additional work and advance planning. A separate kernel must be built to run when the crash dump capability is desired. This kernel needs to be as small as possible, and it must be specially configured to load into a portion of memory not used by the primary kernel. This kernel is also set up so that it only uses a small piece of the total system memory; it must be able to boot and run without changing the primary kernel's memory. 

When the system is running, kexec is used to preload the crash dump kernel into its reserved portion of memory. If all goes well, it simply sits there, wasting memory, and never gets run. That overhead is simply the price one pays for running an enterprise-class kernel. 

Should the system panic, however, the crash dump kernel has its day. The primary kernel, once it decides that something has gone drastically wrong, must first make a copy of the very bottom part of memory (it will get stepped on in the booting process). Once that is done, kexec is invoked to boot directly into the crash dump kernel. That kernel starts up, aware of all memory in the system, but only using the small portion which was reserved to it before. The result is a full, running Linux system with complete access to the memory state of the failed kernel. 

To help with debugging of kernel crashes, the crash dump kernel provides a couple of mechanisms for inspecting the failed kernel's memory. The file `/proc/vmcore` provides the old kernel's memory as an ELF-format core dump; it can be looked at with `gdb` or another debugging tool. If need be, a char device (`/dev/oldmem`) can also be set up; it provides raw access to the old kernel's memory. 

A developer might choose to work with the crash dump kernel and try to track down the problem immediately. In most deployed situations, instead, the crash dump kernel may be configured to simply copy the old kernel's memory image to a disk file somewhere, then reboot back into the primary system. The crash dump file can then be examined at leisure, perhaps by remote support staff. 

The end result of all this work should be a mechanism which can be used to track down the cause of infrequent crashes at remote sites. That should be good for the stability of the kernel as a whole - and the bottom line of enterprise support companies. See [Documentation/kdump.txt](<https://lwn.net/Articles/108596/>) from the patch for more information.
