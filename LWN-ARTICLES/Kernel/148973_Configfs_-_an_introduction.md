---
title: "Configfs - an introduction"
url: https://lwn.net/Articles/148973/
date: "August 24, 2005"
category: Configfs
author: 
---

Complicated kernel subsystems can require complex configuration. Traditionally, Unix-like subsystems have made this configuration possible either via new system calls, or by way of a complex, `ioctl()`-based interface. Neither approach is considered to be optimal. New system calls clutter the namespace and must be added separately for each architecture; they are also quite inflexible once defined and used by user-space code. Anybody who uses the `ioctl()` interface for new code tends to get sneered at; using `ioctl()` is like adding new system calls but without the clear definition of the interface that a system call gives you. 

So how _should_ a new subsystem allow for configuration from user space? In some cases, sysfs can be used. Sysfs, however, was never really meant for this application. It provides a view into the kernel's data structures, and it can be used to cause things to happen with those structures. But sysfs cannot be used to create new objects - at least, not without distorting the interface somewhat. It is the wrong tool for this job. 

The right tool might turn out to be a thing called configfs. It is yet another virtual filesystem, but one which is oriented toward user-space configuration tasks. It is currently part of the OCFS2 patch set, but it is likely to be merged separately due to interest from other kernel projects. It could, conceivably, be merged as early as 2.6.14. 

Configfs is meant to be mounted on `/config`. Each subsystem which uses configfs then creates one or more top-level directories within configfs for their configurations; the distributed lock manager code, for example, creates `/config/dlm/`. That directory can start out empty, or it can be populated with the initial configuration of the subsystem, whichever is appropriate. 

Like sysfs, configfs uses directories as the way of representing objects. Directories contain files ("attributes") which display the current state of the object, and which, optionally, may be writable to change that state. A fundamental difference, however, is that a suitably-privileged user-space process can create directories within configfs. That action will result in a callback within the kernel and the creation of the corresponding object. Directories created within configfs will have a set of attribute files from the beginning. 

As an example (taken from the configfs documentation), consider a hypothetical network block device driver called "fakenbd." This driver would set up `/config/fakenbd`, which would start out empty. A system administrator could then use `mkdir` to create a network disk by creating an appropriately-named subdirectory under `/config/fakenbd`. That directory (called `disk1`, say) would be populated by the kernel with the relevant attributes: `target` for the IP address of the server providing the disk, `device` for the device on the server, and `rw` to control whether the disk is to be writable or not. The administrator would simply write the appropriate value into each attribute, and the disk would be configured. 

Some observers have [questioned](<https://lwn.net/Articles/148978/>) the distinction between configfs and sysfs. Users may well wonder why there are two separate directory trees performing similar tasks - especially since sysfs can be used for certain types of administrative functions. Configfs also has certain problems (such as persistence of attribute permissions) which have already been encountered - and solved - in sysfs. The kernel developers do see the two as being fundamentally different, however, so a merger seems unlikely. 

If configfs takes off, one could imagine it being used all over the kernel. Much of what is done with `ioctl()` now could be moved over. Other patches (such as [CKRM](<http://lwn.net/Articles/145135/>)) which have their own configuration filesystems could switch to configfs. In the long term, configfs could be the path to a much more consistent - and transparent - way of configuring the many subsystems which make up the Linux kernel.
