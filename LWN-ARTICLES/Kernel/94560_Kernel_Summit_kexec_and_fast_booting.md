---
title: "Kernel Summit: kexec and fast booting"
url: https://lwn.net/Articles/94560/
date: "July 21, 2004"
category: Execute in place; Fast booting; Kexec
author: 
---

This article is part of LWN's [2004 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2004/>).   
---  
Randy Dunlap got elected as the leader of a session on kexec and fast booting. There are several approaches to speeding the Linux boot process; they include: 

  * Bypassing the firmware, particularly on reboots. That, of course, is what the kexec patch is for; it allows one kernel to boot directly into another without passing through the BIOS first. 

  * Parallelizing device initialization. The 2.6 kernel still probes for devices in a serial manner; this probing can be slow, but there is usually no real reason why probing for and initializing other devices cannot be happening at the same time. 

  * A known devices database which could eliminate much probing altogether. In many cases, the hardware configuration almost never changes; the startup process could take advantage of that fact and avoid most of the probing it currently does. 

  * Running initialization scripts in parallel. In practice, this technique helps less than one might expect; what is gained in parallelization tends to be lost in disk seeks. 

  * Early user space (initramfs and such) could be used to trim the initialization process to the bare minimum. 

Additionally, the consumer electronics people are interested in "execute in place" (XIP) capability. On many small devices, the kernel (and applications too) is located in flash memory which is directly addressable by the processor. Rather than copy the kernel from flash to RAM, why not run it directly from its place in flash, avoid the copy, and preserve RAM space? The downside is that flash tends to be slower, so there will be a performance penalty. There is talk of trying to copy just the "hotspots" in the kernel code to RAM; this approach looks like a recipe for complexity and trouble. 

The kexec patch turns out to have another use, which was revisited in the next session: it can be set up to preserve part of kernel memory over the reboot. If things are carefully done, it can thus be used to implement a sort of crash dump capability. 

[>> Next: RAS tools](<https://lwn.net/Articles/94561/>).
