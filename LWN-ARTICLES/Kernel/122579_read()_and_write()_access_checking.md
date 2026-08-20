---
title: read() and write() access checking
url: https://lwn.net/Articles/122579/
date: "February 8, 2005"
category: Security; System calls
author: 
---

Long ago, when the 2.0 kernel was the state of the art, the implementation of the `read()` and `write()` system calls (and `readv()` and `writev()` too) behaved a little differently than now. Then, as now, the main purpose of the core implementation of those system calls was to pass the call on to the appropriate function in the filesystem code or device driver handling the file of interest after dealing with any relevant file locking details. In many ways, `sys_read()` and friends in 2.6 look very much like they did in 2.0. 

The 2.0 implementation differed, however, in that it checked whether the calling process had the ability to read or write the buffer it passed into the kernel. The semantics of a `read()` call, say, should be the same regardless of where the data is being read from. So it made sense to check, before invoking the VFS or device driver, that the buffer passed to `read()` was writable by the calling process. In 2.2, that check went away, possibly as part of the big changes made to how user-space access checks were implemented. Performing those checks became entirely the responsibility of the lower-level code. 

Linus recently [merged a patch](<https://lwn.net/Articles/122581/>) which restores the upper-level checks for 2.6.11. The reason given with the patch is that checks performed in lower-level code only verify the range of memory which will actually be read from or written to. If that range is smaller than the application requested (because the file is not that long, say), part of the range requested by the application will not be checked. The operation of the system is entirely correct in this case, but an opportunity to flag a bug in the calling program will have been missed. 

It also doesn't hurt that placing the check at the entry point to the kernel ensures that it will be done in all situations. One less opportunity for security problems resulting from forgotten checks in lower-level code can only be a good thing. It seems almost certain that at least one such vulnerability must exist somewhere in the 2.6 kernel. 

One might conclude that low-level code, such as device drivers, need no longer perform the `access_ok()` check, since it is now being handled at a higher level. A prudent developer, however, would probably leave that check in place. It is quite cheap on most architectures (it generally just ensures that the given buffer is not located in kernel space), and the higher-level checks went away once before. Safe is better than sorry, especially when being safe is so easy. 

(For completeness, it's worth noting that Linus merged [another patch](<https://lwn.net/Articles/122590/>) which ensures that a read or write operation does not overflow the file offset).
