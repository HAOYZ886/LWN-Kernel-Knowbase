---
title: Kernel headers and user space
url: https://lwn.net/Articles/113349/
date: "November 30, 2004"
category: Coding style; Include files
author: 
---

The use of kernel headers in user space has long been discouraged. The kernel headers are not written with user space in mind, and they can change at any time. The proper way for user-space applications to interface with the kernel is by way of the C library, which provides its own structures and, when necessary, translates them into whatever the current kernel expects. This separation helps to keep user-space programs from breaking when the kernel changes. 

Unfortunately, things do not always work that way, and some user-space programs still end up including kernel headers directly. These programs may simply be old, or they may need access to declarations which are not available in the C library include files - strange `ioctl()` codes, for example. So the kernel code still tries to make it possible for user space to include some header files. In these files, kernel-specific code is contained within `#ifdef __KERNEL__` blocks and hidden from user space. This technique works, but it is brittle and adds extra cruft to the kernel code base. Intermixing internal kernel definitions with those needed by user space also makes it easy to break the user-space API. 

The kernel developers have, for years, wanted to improve this situation. The latest attempt came in the form of [this RFC](<https://lwn.net/Articles/113006/>) from David Howells. This proposal would create some new directories in the kernel source tree: `include/user` and some architecture-specific variations (such as `include/user-i386`). When a portion of a kernel header file is found to be needed by user space, it would be placed into a separate file in one of those directories, and the new file would be included into the old one. At this point, the definitions needed by user space will have been separated out, but no visible changes will have been made; user space can still include the old file and get what it needs. 

At some future point, when user space is deemed to have been fixed, all of the `__KERNEL__` references could be removed from the old files. At that point, any application still including the internal header files would break. 

One part of the idea which did not get very far was using standard C types (such as `uint16_t` and such) for the user-kernel interface. The problem with that idea is that the kernel cannot count on those types being consistently defined for all configurations, and cannot create its own definitions for the standard types. So the kernel/user interface must continue to be defined using kernel-specific types (`__u16` and such). 

Linus was [not all that enthusiastic](<https://lwn.net/Articles/113367/>) about the idea in general. To him, it looks like an exercise in rearranging things without specific goals and with the possibility of breaking things which work now: 

We undeniably have existing users of kernel headers. That's just a fact. If we break them, it doesn't _matter_ how the kernel headers look, and then "existing practice" is about as good an organization as anything, and breaking things just to break things is not something I'm in the least interested in. "Beauty" comes secondary to "usefulness". 

What he would like to see is more specific discussions which identify specific, problematic header files and what will be done to fix them. In the end, the header files might just get split up in the way described by Mr. Howells. It is more likely to happen as a long and slow process, however, and not as a massive, coordinated reorganization.
