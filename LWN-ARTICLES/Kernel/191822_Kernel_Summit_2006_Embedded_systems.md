---
title: "Kernel Summit 2006: Embedded systems"
url: https://lwn.net/Articles/191822/
date: "July 19, 2006"
category: Embedded systems
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
Matt Mackall led a relatively uncontentious session on the state of Linux for use in embedded systems. There are a few outstanding issues which need attention if Linux is to become more useful in this field. 

One of those is filesystems for modern, large flash drives - filesystems like JFFS2 were never meant to scale up to that size. Current devices are pushing the limits, and there is pressure from the One Laptop Per Child project as well. Work is in progress to come up with an alternative, but details are vague at this point. 

Boot time is said to be getting steadily worse, and there is not a whole lot of work going into improving the situation. It was suggested that user space tends to be the larger part of the problem, but, on embedded systems, that is often not the case. Embedded systems vendors have a fair amount of control over their user spaces, and they have optimized that part of the system to the greatest extent that they can. Fixing the kernel side of the equation is a harder task, however. Much of the kernel's boot time is spent probing for devices, and there are things which can be done to improve that process. Probing in parallel is the obvious first step, though parallel probing does open the door to a lot of unpleasant race conditions. It is a solvable problem, however. 

Thomas Gleixner talked a bit about timers. In particular, the embedded systems vendors have a real need for the [dynamic tick](<http://lwn.net/Articles/149877/>) code. This patch, often called "dyntick," does away with the regular clock tick in favor of explicitly scheduled timer interrupts when the kernel determines that it will need to be awakened. Eliminating the clock tick cuts power consumption, especially in situations where the CPU would otherwise be sleeping. Thomas noted that many systems can, with dyntick, run with about 1.5 actual ticks per second - a significant reduction from the default 250Hz clock rate. But one must first kill the GNOME clock applet, which, for some reason, requires a 100Hz tick of its own. 

Linus noted that the ARM architecture has offered a dynamic tick feature for some time. The current version of the dyntick patch differs significantly from the ARM implementation, but the ARM maintainers are evidently willing to move over to it. So there is no real reason for keeping dyntick out of the kernel. 

Matt talked briefly about kernel code size. It turns out that, in recent times, the size of the i386 kernel has actually gone down slightly. That reduction is mainly the result of Matt's work in shrinking code, eliminating unnecessary inline functions, and providing replacement code (such as the SLOB allocator) for kernel subsystems whose complexity is unneeded in an embedded environment. 

There was no discussion of one of the ongoing problems in this area: getting embedded systems developers to participate in the community and contribute improvements back to the mainline kernel. 

  * [Next: Security](<http://lwn.net/Articles/191737/>).
