---
title: "Kernel Summit 2006: The user-space API"
url: https://lwn.net/Articles/191654/
date: "July 18, 2006"
category: "User-space API; klibc"
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
As the interface presented to user space by the kernel grows in complexity, keeping that API stable is getting harder. Greg Kroah-Hartman led a session on this API and what can be done to improve things in this area. The first thing Greg pointed out is the `Documentation/ABI` directory in the kernel source, which is meant to be the definitive description of the kernel interface. Once an interface is described there, the kernel is supposed to implement it that way, and, crucially, not change it. It was suggested that this documentation could be cited by users as a way of complaining about inadvertent API changes, but Linus said it wouldn't work that way. Documentation, he says, is always wrong; it's what the kernel actually does that matters. 

With regard to stability, it was noted that `ioctl()` calls tend to remain stable, while, for example, `/proc` and sysfs interfaces have had a distressing tendency to change. Some thought that the assignment of an operation number somehow had the effect of impressing on developers that changes could not be made. There are, however, a couple of other, more plausible arguments that were made: 

  * Sysfs directly couples the kernel ABI to the internal data structures used internally. Kernel developers have never accepted limits on the changes they could make internally, but, with sysfs, all of those changes are reflected in the ABI - sometimes without the developer's knowledge. Over time, this coupling is likely to be loosened. 

  * The Linux system call interface is, for the most part, an implementation of a well-established standard. That standard gave little latitude in how the user-space API could work. With interfaces like sysfs, however, the developers are in uncharted territory. So it is not surprising that more mistakes are made in the creation of the API, and that it tends to be a little more fluid. 

Greg also stated that many of the problems with sysfs are the result of errors on the user-space side. Even Greg, in the development of `udev`, got things wrong. When asked whether this tendency toward errors might indicate that the sysfs interface was inherently unsafe, he replied in the negative. It is hoped that the newer ABI documentation will make the rules for working with sysfs more clear and less error-prone. 

The discussion moved on to klibc and whether it should be a part of the kernel. Linus replied that klibc is not interesting by itself - it's more a matter of what interesting things could be done with it. And those interesting things are yet to arrive; Linus is unconvinced by recent efforts to move large chunks of code into user space. There is usually no point, he says, in moving working code out of the kernel, and, in any case, anything which involves shared state should stay in the kernel. So the idea of moving much of the early boot process into initramfs might not get very far. 

Fears were expressed that, by putting more of the kernel into a user-space initramfs area, the developers could create a situation where users are tied to their distributor's kernel. There is a strong desire to ensure that mainline kernels continue to work on as many distributions as possible. 

One question which comes up occasionally is whether code which is tightly coupled to the kernel should be packaged with the kernel. A common example is `udev`, which has been broken by a kernel release at least once. If tools like `udev` lived with the kernel, the tools and the kernel could be updated together and version skew problems would not develop. There is some sympathy toward this idea, but no immediate plans to add any tools to the kernel source tree. 

When such plans are made, kernel crash dump tools might be at the top of the list. Another possibility is the `hal` program used by the GNOME and KDE desktops. It was described as "unmaintained crap" which is heavily depended upon by the desktop environments. The kernel developers, it was suggested, should help to get `hal` onto a better footing, possibly by bringing it into the kernel source tree. 

  * [Next: Software suspend](<https://lwn.net/Articles/191657/>).
