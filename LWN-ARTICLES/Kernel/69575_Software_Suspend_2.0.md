---
title: Software Suspend 2.0
url: https://lwn.net/Articles/69575/
date: "February 4, 2004"
category: Software suspend
author: 
---

The better part of a year ago, your editor replaced his ancient Sony Vaio laptop with a new Vaio laptop. The new machine is quite nice in many ways, but it came with an interesting surprise: the old BIOS-based suspend-to-disk functionality was no more. In the modern world, suspending the system is supposed to be done by the operating system, not by the hardware; that's what we call "progress." 

Ever since getting the new laptop, your editor has been interested in the software suspend patch, which promises to restore that missing functionality. Versions of that patch have been working reasonably well for a while, but software suspend work has not stood still. The [announcement](<https://lwn.net/Articles/68844/>) of the software suspend 2.0 patch was thus of interest. 

The new patch brings with it a number of new improvements. Software suspend now works on systems with high memory (up to 4GB, which will be sufficient for most laptops for a little while yet), SMP systems (2.4 only), and preemptive kernels. Suspend-to-disk will now work with swap files, not just dedicated partitions. Compression of the saved image is supported, which can lead to faster suspends and resumes on some systems. And, of course, there is a nicer, splash-screen enabled user interface. 

The fact remains, however, that software suspend is a hard problem, and the Linux version still has some ground to cover before it is truly ready for general use. Your editor had no end of trouble getting the 2.0 patch to work until the software suspend hackers pointed out the USB code which had been built into the kernel. USB and power management do not yet play very well together, it seems. The only way to make the 2.0 patch work reliably on systems with USB is to compile all of the USB code in modular form so that it may be removed from the kernel prior to suspending. There are also issues with AGP video, SMP under 2.6, and various other parts of the system. Software suspend can be made to work well, but you have to be prepared to dig into the kernel a bit to get there. 

It is encouraging to see how quickly this work is proceeding, however. A stable, safe, reliable software suspend functionality later in the 2.6 series could well come about. (If you are interested in how software suspend works, see [the May 1, 2003 LWN Kernel Page](<https://lwn.net/Articles/30098/>)).
