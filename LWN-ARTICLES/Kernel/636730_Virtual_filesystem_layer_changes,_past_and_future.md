---
title: "Virtual filesystem layer changes, past and future"
url: https://lwn.net/Articles/636730/
date: "March 16, 2015"
category: "Filesystems-Virtual filesystem layer; iov iter"
author: "By Jonathan Corbet March 16, 2015 LSFMM 2015"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 16, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015>)

While most of the 2015 Linux Storage, Filesystem and Memory Management summit was dedicated to subsystem-specific discussions, some subjects were of sufficiently wide interest that they called for plenary sessions. Al Viro's session about the evolution of the kernel's virtual filesystem (VFS) layer was one such session. There is little that happens in the system that does not involve the VFS in one way or another; in a rapidly changing kernel, that implies a need for the VFS to change quickly as well. 

One of the things that has not yet happened, despite wishes to the contrary, is the provision of a better set of system calls to replace `mount()`. Al did some work in that area but the patches got bogged down before they were even posted for review. So there is no real progress to report in that area yet. On the other hand, there has been some limited progress toward the creation of a `revoke()` system call. The full implementation remains distant, but some of the infrastructure work is done. 

An area that has seen more work is the transition to the [`iov_iter` interface](<https://lwn.net/Articles/625077/>). Al's hope is that, by the time the 4.1 merge window closes, the reworking of `aio_read()` and `aio_write()` (part of the asynchronous I/O implementation) to use `iov_iter` will be complete. There are several instances that still need to be converted, but he is reasonably confident that there are no significant roadblocks. 

In the last year the send and receive paths in the network stack have seen `iov_iter` conversions. The `sendpages()` path remains to be done, but there do not seem to be any obstacles to getting it done. The conversion of the [`splice()`](<https://lwn.net/Articles/178199/>) system call is a bit harder. The code on the write side has almost all been switched, with one exception: the filesystem in user space (FUSE) module. The problem with FUSE is that it wants to do zero-copy I/O, moving pages directly between a `splice()` buffer and the page cache. 

When `splice()` was first added to the kernel, this sort of "page stealing" was part of the plan; it seemed like a useful optimization. But page stealing had a number of problems, including confusion in the [![\[Al
Viro\]](https://static.lwn.net/images/conf/2015/lsfmm/AlViro-sm.jpg)](<https://lwn.net/Articles/636741/>) filesystem code when an up-to-date page is stuffed directly into the page cache. So Nick Piggin [removed that feature](<http://git.kernel.org/linus/485ddb4b9741bafb70b22e5c1f9b4f37dc3e85bd>) in 2007 and nobody has ever gotten around to putting it back. Al noted that Nick described some of the problems in his commit message, but there are others and, since Nick has proved hard to reach in recent years, they will have to remain a mystery until somebody else rediscovers them. 

Meanwhile, zero-copy operation in `splice()` is disabled, with one exception: FUSE. The problems that affected page stealing with other filesystems do not come up with FUSE, so there was no reason to disable it there; beyond that, FUSE needs zero-copy operation or its performance will suffer. This has prevented the conversion of FUSE over to `iov_iter` for now. Al's preferred solution to this problem would be to restore the zero-copy mode for all cases, but that is going to take some exploration. 

The read side (as represented by the `splice_read()` `file_operations` method) will probably be converted sometime this year. 

In summary, Al said, he is surprised by how many `iovec` instances (the predecessor to `iov_iter`) remain in the kernel. It is not about to go extinct quite yet, but there are fewer and fewer places where it is used. 

Another upcoming change that might be visible outside of the VFS is that the `nameidata` structure is about to become completely opaque. It will only be defined within the VFS code. Al would like to eventually get rid of even the practice of passing around pointers to this structure and switch to using a pointer out of the task structure. This change should not affect non-VFS code that much, but he wanted to mention it because there are patch sets out there that will be broken. 

Work continues on the project of getting rid of the numerous variants of `d_add()`, the basic function that adds a directory entry (`dentry`) structure into the dentry cache. One of those variants — `d_materialise_unique()` — was removed in 3.19. Others, like `d_splice_alias()`, remain. The ideal situation would be to have a single primitive to associate dentries with inodes. Matthew Wilcox asked if the other variants might still have value for documentation purposes, but Al said such cases should be handled with assertions. 

A couple of other recent changes include unmounting of filesystems on invalidation and better shutdown processing. The unmounting changes cause a filesystem to be automatically removed if its mount point is invalidated; it went in some months ago. The big change with filesystem shutdown processing is that it is now delayed and always run on a shallow stack. That should address concerns about stack overflows that might otherwise occur during shutdown processing. 

Al's final topic had to do with BSD process accounting. What happens if you start accounting to a file, then unmount the underlying filesystem? On a BSD system, the unmount will fail with an `EBUSY` error. But, on Linux, "somebody decided to be helpful" and thought it would be a friendly gesture to automatically stop the accounting and allow the unmount to proceed. This policy seems useful, but there is a catch: it creates a situation where an open file on a filesystem does not actually make that filesystem busy. That has led to a lot of interesting races dating back to 2000 or so; it is, he said, a "massive headache." 

This mechanism has now been ripped out of the kernel. In its place is a mechanism by which an object can be added to a `vfsmount` structure (which represents a mounted filesystem); that object supports only one method: `kill()`. These "pin" objects hang around until the final reference to the `vfsmount` goes away, at which point each one's `kill()` function is called. It thus is a clean mechanism for performing cleanup when a filesystem goes away. 

The first use of this mechanism is to handle shutdown of BSD process accounting. But it can also be put to good use when unmounting a large tree with multiple filesystems. If one filesystem depends on another, a pin object can be placed to ensure that the cleanup work is done in the right order. This facility, found in [`fs/fs_pin.c`](<https://lwn.net/Articles/636740/>) looks to be useful but, as Ted Ts'o noted, it is also completely undocumented at the moment. Al finished the session with an acknowledgment that some comments in that file would be helpful for other users.
