---
title: "Kernel Summit 2005: The Kernel and the Linux desktop"
url: https://lwn.net/Articles/144279/
date: "July 20, 2005"
category: Desktop support
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
Jon Smirl, Dave Airlie, and Keith Packard took time away from the Desktop Developers' Conference to talk about what the kernel can do to help improve the Linux desktop. The core of this discussion - cleaning up the mess of code which deals with driving video cards - mirrors last year's discussion, but some things have come into focus since then. 

The Linux virtual terminal subsystem was designed in the days of simple VGA adaptors. Today's hardware can have hundreds of (undocumented) configuration registers and 1GB of video RAM. What's more, a compositing X server can actually _use_ all of that RAM. 

A big part of the problem, it turns out, is the longstanding ability to change between virtual consoles at will. Saving and restoring the frame buffer state gets harder as the hardware gets more complicated. And saving that 1GB of VRAM can take several seconds - rather longer than most users are expecting to wait. Then throw in modern, multi-head video cards which might be in simultaneous use by several users and everything gets difficult. 

So the desktop developers came in with a proposal that the old virtual console switching capability should be tossed out. Instead, the frame buffer should always operate in one of two graphical modes: a simple "system console" mode and the full graphics mode. This particular room, however, probably contained the highest concentration of virtual console users on the planet, so that idea was not entirely popular. Linus proposed, instead, that the current virtual console modes should be kept as a "legacy" way of doing things. When the newer graphics code gets control of the console, instead, it can do things its way - including refusing to switch back to the old mode if need be. Meanwhile, existing software - and usage habits - will continue to work. 

Dave Airlie got into the details of how the kernel and the X Window System should interact. In the end, the kernel should be responsible for the low-level interaction with the hardware. At the least, this involves basic interaction with the bus and the device. It is not currently possible, however, to perform all mode setting from within the kernel. The kernel drivers do not have all of the needed capabilities (dual head support, TV out configuration, etc.), and the X drivers are far better debugged. So the kernel may handle some low-level tasks, including acceleration control and VRAM memory management, but the rest will be handled from user space. In any case, the control interface will be well separated from rendering. 

One other nagging issue is suspend and resume; that one was deferred to the power management session. 

Keith Packard reported briefly that a "credible demo" of a fully GL-based X server has been making the rounds. The performance is impressive, as are the eye candy possibilities.
