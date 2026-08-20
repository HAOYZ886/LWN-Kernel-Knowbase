---
title: Initramfs arrives
url: https://lwn.net/Articles/14776/
date: "November 6, 2002"
category: initramfs
author: 
---

The "initramfs" concept has been in the 2.5 plans since back before there _was_ a 2.5 kernel. Things have been very quiet on the initramfs front, however, until [the first patch](<https://lwn.net/Articles/14448/>) showed up and was merged into the 2.5.46 tree. 

The basic idea behind initramfs is that a cpio archive can be attached to the kernel image itself. At boot time, the kernel unpacks that archive into a RAM-based disk, which is then mounted and used at the initial root filesystem. Much of the kernel initialization and bootstrap code can then be moved into this disk and run in user mode. Tasks like finding the real root disk, boot-time networking setup, handling of initrd-style ramdisks, ACPI setup, etc. will be shifted out of the kernel in this way. 

An obvious advantage of this scheme is that the size of the kernel code itself can shrink. That does not free memory for a running system, since the Linux kernel already dumps initialization code when it is no longer needed. But a smaller code base for the kernel itself makes the whole thing a little easier to maintain, and that is always a good thing. But the real advantages of initramfs are: 

  * Customizing the early boot process becomes much easier. Anybody who needs to change how the system boot can now do so with user-space code; patching the kernel itself will no longer be required. 

  * Moving the initialization code into user space makes it easier to write that code - it has a full C library, memory protection, etc. 

  * As [pointed out](<https://lwn.net/Articles/14780/>) by Alexander Viro: user-space code is required to deal with the kernel via system calls. This requirement will flush a lot of in-kernel "magic" currently used by the initialization code; the result will be cleaner, safer code. 

The patch, as found in 2.5.46, does not do a whole lot; it adds the basic mechanism but only removes "three simple lines" from the current initialization code. The bulk of the code will be added in the coming weeks - now that the "feature" is in the kernel, the details can be filled in without, technically, breaking the feature freeze. The plan for those steps has been laid out by Jeff Garzik: 

  * A small C library ("klibc") will be merged to support initramfs applications. 

  * A small "kinit" application will be created with klibc. In the beginning, it will only do enough work to show that the mechanism is functioning properly. 

  * The "initrd" (initial ramdisk) subsystem will be moved into kinit, and out of the kernel itself. 

  * The mounting of the root filesystem will be moved to user space. A lot of code for dealing with things like NFS-mounted root filesystems will go away. 

That is as far as the plan goes, for now. There is no doubt that other parts of the initialization process will be moved to user space, however; it will be interesting to see how that process goes. 

There a couple of fundamental open questions that will have to be answered during the remaining 2.5 development period. One is whether the initialization process should be handled by a single "kinit" application, or whether it should be a collection of programs, and, probably, shell scripts. Then, there is the question of what to do with klibc. It will be packaged with the kernel for now, but a number of kernel developers think that klibc (and the whole user-space initialization setup) should eventually be split off into a separate project. These decisions might not be made until very shortly before the stable release.
