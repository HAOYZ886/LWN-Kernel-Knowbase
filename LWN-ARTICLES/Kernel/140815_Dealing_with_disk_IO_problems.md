---
title: Dealing with disk I/O problems
url: https://lwn.net/Articles/140815/
date: "June 21, 2005"
category: Events reporting
author: 
---

Filesystem authors try hard to avoid losing data. Many of them have discovered, the hard way, that failure to return a user's bits in exactly the same condition as when they were entrusted to the filesystem can lead to serious disgruntlement down the road. There are limits to what a filesystem can do, however, when the hardware starts to fail. If a disk drive begins to go bad, or somebody yanks out a hotpluggable device, problems are simply going to happen. 

So what should a filesystem do in such a case? The behavior shown by most Linux filesystems (and partially enforced by the VFS layer) is to return an I/O error status (`EIO`) when things start to fail, then remount the filesystem in a read-only mode in an attempt to avoid any further damage. The end result is that a user-space application _might_ see an `EIO` error return once - or it might not, since not all in-kernel error codes make it all the way back to user space. After that, the returned error will be `EROFS` (read-only filesystem), which is not entirely illuminating. 

Back in the good old days, we would just look in the system log file to see what was really going on. The new crowd of Linux users would rather not have to do that, however; they expect the system to tell them, politely, that their hardware is on fire and that they are about to deeply regret not having run any backups since sometime last winter. The problem is that the POSIX API is simply not set up to return that sort of detailed error information. Breaking compatibility with POSIX is not an option, so something complicated would have to be done to return error information within the bounds of the current API. Beyond that, however, is the simple fact that the application which is currently beating its head against disk errors might not be the right one to be having a pleasant conversation with the user about those errors. 

These issues have led Ted Ts'o to [suggest](<https://lwn.net/Articles/140822/>) that a different mechanism should be used. Rather than try to shove additional information through the existing API, the kernel should simply report events like disk disasters via an out-of-band mechanism. For example, errors could be reported with the user notification mechanism and fed into [DBus](<http://www.freedesktop.org/Software/dbus>) for distribution. The user could then be informed of the trouble and given the opportunity to panic in a desktop-specific manner. 

There seems to be a high level of agreement that the out-of-band notification is the right way of doing things. All that is needed is for somebody to do the hacking to actually make it happen.
