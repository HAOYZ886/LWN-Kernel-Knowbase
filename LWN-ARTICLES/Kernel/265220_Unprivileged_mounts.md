---
title: Unprivileged mounts
url: https://lwn.net/Articles/265220/
date: "January 15, 2008"
category: "Filesystems-Mounting"
author: "By Jonathan Corbet January 15, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 15, 2008

There are a number of filesystem-related patches aimed at the upcoming 2.6.25 merge window; one of those is the [unprivileged mount patch](<http://lwn.net/Articles/264188/>) by Miklos Szeredi. This patch enables an unprivileged user process to call the `mount()` system call and - in certain circumstances - have that call actually succeed. It could eventually lead to a situation where users have more flexibility to create their own environments and the setuid `mount` utility is no longer needed. 

This patch adds a new field (`uid`) to the `vfsmount` structure, allowing the kernel to keep track of the owner of a specific filesystem mount. The system administrator can give ownership of a specific mount to a user with the new `MNT_SETUSER` flag. A common pattern might be to bind-mount a user's home directory on top of itself, giving the user the ownership of that mount. Once that has been done, the user is allowed to freely mount other filesystems below that mount point - with a couple of conditions: 

  * There is a system-wide limit on the number of allowed user mounts; once that limit is hit, no more unprivileged mounts will be allowed until somebody unmounts something. The current patch has no provision for per-user or per-group mount limits, but such a feature would not be particularly hard to add should the need arise. 

  * The filesystem type must be marked as being safe for unprivileged mounts. Miklos notes that a filesystem must go through "a thorough audit" before this flag can be set with any confidence. The patch, as posted, marks the fuse filesystem (which allows for the creation of filesystems implemented in user space) as being safe; fuse was designed for this mode of operation in the first place. Bind mounts are also allowed, with some additional conditions. 

If the system allows the mount, the flags allowing for setuid and device files will be forcibly cleared - unless the user has the requisite capabilities anyway. Users are allowed to unmount filesystems they own, again without privilege, but cannot unmount any others. Another new mount flag (`MNT_NOMNT`) marks a specific filesystem as being the end of the line - no unprivileged submounts are allowed below it. The end result of  [PULL QUOTE:  One might well wonder why this change to the `mount()` system call is called for, given that users have been able to do unprivileged mounts for years.  END QUOTE] all this should be a mechanism by which users can organize their filesystem hierarchies without any need for administrative privileges, and without the risk of compromising system security. 

One might well wonder why this change to the `mount()` system call is called for, given that users have been able to do unprivileged mounts for years. The answer is that the current mechanism has a couple of shortcomings. Every potential unprivileged mount must be explicitly enabled via a line in `/etc/fstab`. That works well for simple situations, such as allowing a user to mount a CD or a USB storage device. When users start wanting to do more complicated things, like mounting their own special fuse filesystems, the `/etc/fstab` mechanism breaks down. There is a separate, setuid program which grants the right to make unprivileged fuse mounts, but it represents a workaround rather than a proper solution. 

The current user mount mechanism also requires that the `mount` utility be installed setuid root. Every setuid binary is a potential security hole, so there is value in eliminating privileged programs when possible. The unprivileged mount patch offers the possibility of eliminating the setuid mount program while simultaneously leaving policy control in the hands of the system administrator. So, unless something surprising comes up, chances are good that this capability will appear in the 2.6.25 kernel.
