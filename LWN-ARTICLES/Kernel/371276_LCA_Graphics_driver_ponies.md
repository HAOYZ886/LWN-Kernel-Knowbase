---
title: "LCA: Graphics driver ponies"
url: https://lwn.net/Articles/371276/
date: "January 26, 2010"
category: Device drivers; Nouveau
author: "By Jonathan Corbet January 26, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 26, 2010

Those of you who have come to appreciate Dave Airlie's kitten-filled presentations might just have been dissatisfied with his linux.conf.au talk, which was called "So you moved graphics drivers to the kernel.. what next? I can haz ponies?" But ponies, too, can be cute, and the update on the state of graphics drivers in the kernel was well worth the listening. 

It has now been about a year since kernel mode setting (KMS) was merged into the mainline kernel. KMS ends the "mess" which came from having graphics drivers in user space; digging out of that particular hole took a good seven years or so. But now our graphics drivers are in the kernel, just like most other drivers. 

Beyond cleaning up the mess, there are a few other good reasons for merging KMS. One is that the system is now able to make full use of the power-saving features of the hardware; before KMS, the kernel never really knew enough about what was going on with the hardware to do this. The Intel drivers can now perform as well as Windows with regard to power saving; the ATI drivers, instead, are not quite there yet. Another nice [![\[Dave
Airlie\]](https://static.lwn.net/images/conf/lca2010/DaveAirlie-sm.jpg)](<https://lwn.net/Articles/371302/>) feature is the ability to use a kernel debugger on a system with graphics running; it's now possible to trap into the debugger, then return to a running system and have everything just work. 

One of the reasons why KMS took so long to merge is that it places a number of new requirements on the kernel. At the top of the list is a proper manager for graphical memory. That's a hard problem, one that the graphics developers always intended to get to sometime Real Soon Now. Eventually the [TTM](<http://lwn.net/Articles/257417/>) developers got to it, but they quickly ran into a number of API difficulties. After some effort, the Intel developers decided that a generic approach to the memory management API wasn't going to work; out of that realization came the GEM memory manager, which only tried to solve the Intel problem. 

Developers working on ATI chipsets, in turn, soon realized that GEM did not have the capabilities that they needed. So they went back to TTM, but not before bolting something that looks a lot like the GEM API onto it. TTM was recently merged, making KMS possible for ATI chipsets as well. 

So what is coming? One future feature is the [Gallium](<http://wiki.freedesktop.org/wiki/Software/gallium>) 3D architecture. Gallium, says Dave, is starting to work, but full functionality will take a while yet. Moving drivers to Gallium is going to be a painful exercise; there are already plenty of APIs that these drivers need to support. [DRI2](<http://www.x.org/wiki/DRI2>) is also coming along. This feature really needed KMS to work properly, especially when compositing is being used. There are still performance issues which must be resolved, though. 

Another thing to look forward to is the [Wayland](<http://groups.google.com/group/wayland-display-server>) display server. Wayland can be seen as a simpler, smaller replacement for X built on KMS. It can run GTK and GL applications now; there is also an X server emulator which can run on top of it. A few difficulties remain, including the fundamental fact that Wayland is not X; since X is the standard in this area, alternatives are going to be hard to sell. The Wayland developers also have not yet really dealt with the input problem, but input is a big piece of the X code. So Wayland, too, will be a while in coming; it may find its way into embedded situations first. 

Dave spent some time on the current state of the graphics drivers. **Intel** , he says, is currently in the leading position. It supports KMS for everything - well, almost everything; the "chipset we won't name" (the proprietary GMA500) still lacks support. The driver is feature-complete, but Dave isn't quite ready to call it "mature"; another release or two will be required first. As [discussed here](<http://lwn.net/Articles/369706/>) previously, the driver will need to retain user mode setting (UMS) support for some time, but the current upstream X.org sources have already removed UMS from the X server. 

The **ATI/AMD** drivers are further behind, but getting closer; this driver is harder than the Intel driver, due to the large number of chipset variations. Chipsets from R100 to R700 are currently supported; R800 support can be expected within a few weeks. The driver works "nearly as well as the old stuff" at this point; suspend and resume work better than before. Support for power-saving features is missing but expected for 2.6.34. The Radeon driver is currently in the staging tree, but it might move out before the end of the 2.6.33 development cycle. 

What about the RadeonHD driver? That fork of the driver is primarily the result of a disagreement over the use of ATI's BIOS tables; the Radeon driver has an interpreter for these tables, while RadeonHD reimplements the functionality that those tables provide. Using the BIOS tables makes life a lot easier; it lets the driver ignore a lot of the details associated with different chipset variations. The BIOS table code is part of the KMS implementation which has been merged into the mainline; that should, Dave thinks, resolve this disagreement. 

The "pony" displayed for the **Nouveau** discussion was a Trojan horse. Nouveau, of course, was [merged for 2.6.33](<http://lwn.net/Articles/366648/>). The driver has just lost its user-mode support; it will be KMS only. Chipsets from the NV4 through the G80 are supported, with the final pieces to be filled in soon. The "ctxprogs" firmware is being figured out; the NV40 version has already been replaced with a rewritten, freely-licensed equivalent and NV50 is in the works. Dave noted that, whatever one thinks about NVIDIA's approach to working with the community, its hardware tends to be relatively good and easy to work with. 

When Dave was asked about support for non-Linux systems, he replied that most of them have been left behind at this point. There is, apparently, an OpenSolaris port being done within Sun, but no code has been released from that group. One other audience member asked about running X without root privileges: that does work now, and Moblin is doing it. There are some problems remaining, though, especially with fast user switching. In the absence of a `revoke()` system call, there's no way to guarantee that one user isn't listening in on another. Since `revoke()` is known to be a hard problem, it's not clear how this issue will be resolved.
