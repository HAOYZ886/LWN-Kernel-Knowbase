---
title: "FUSE - implementing filesystems in user space"
url: https://lwn.net/Articles/68104/
date: "January 27, 2004"
category: "Filesystems-In user space"
author: 
---

[Last week](<http://lwn.net/Articles/66289/>) we looked at implementing device drivers in user space. Drivers are not the only kernel functionality which can be moved across the divide, however; it is also possible to implement filesystems with user-space code. Linux has a long tradition of user-space filesystems, actually; NFS was implemented that way for quite some time. Even so, user-space filesystems are not widely used, for a number of obvious reasons (performance, security, ...). But there are situations where a user-space filesystem can be a nice thing to have. 

For those situations, there is a project called FUSE. Its associated [SourceForge page](<http://sourceforge.net/projects/avf>) is not particularly enlightening; one really has to look at the project's code to understand what FUSE has to offer. Since the second FUSE 1.1 release candidate has just been [announced](<https://lwn.net/Articles/68048/>), this seems like a good time for such an examination. 

FUSE is a three-part system. The first of those parts is a kernel module which hooks into the VFS code and looks like a filesystem module. It also implements a special-purpose device which can be opened by a user-space process. It then spends its time accepting filesystem requests, translating them into its own protocol, and sending them out via the device interface. Responses to requests come back from user space via the FUSE device, and are translated back into the form expected by the kernel. 

In user space, FUSE implements a library which manages communications with the kernel module. It accepts filesystem requests from the FUSE device and translates them into a set of function calls which look similar (but not identical) to the kernel's VFS interface. These functions have names like `open()`, `read()`, `write()`, `rename()`, `symlink()`, etc. 

Finally, there is a user-supplied component which actually implements the filesystem of interest. It fills a `fuse_operations` structure with pointers to its functions which implement the required operations in whatever way makes sense. This interface is not well documented, but [the example filesystem](<https://lwn.net/Articles/68106/>) provided with FUSE (which implements a simple sort of loopback filesystem) is reasonably easy to follow. 

An old filesystem module (AVFS) uses FUSE to make filesystems out of tar and zip files, but one could imagine any number of other possibilities. It would not be that hard to make filesystems which mirror a web site (in read-only mode, at least), provide access to an object database, or provide a file-per-user view of the password file, for example. FUSE could be an ideal platform for experimenters who want to take the "everything is a file" idea to its limit.
