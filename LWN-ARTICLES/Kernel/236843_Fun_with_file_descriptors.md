---
title: Fun with file descriptors
url: https://lwn.net/Articles/236843/
date: "June 4, 2007"
category: File descriptors; System calls
author: 
---

[Last week's article on syslets](<http://lwn.net/Articles/236206/>) briefly mentioned a problem with using file descriptors for low-level communications with the kernel. There is a single namespace for file descriptors, combined with a strict rule for how those descriptors are allocated. As long as the application is fully in charge of that space all works well, and the "lowest available descriptor" rule can be relied upon. As soon as hidden levels (the C library in particular) start using file descriptors for their own purposes, though, the potential for conflicts and confusion at the application level arises. An application which makes a mistaken assumption about where a file descriptor will be allocated, or which indiscriminately "cleans up" open descriptors belonging to the libraries will break. This problem is evidently real, to the point that the glibc goes out of its way to avoid using internal file descriptors for anything. 

This issue is a problem for kernel developers. They would rather not create new, file-descriptor-based services (completion events for syslet-based asynchronous I/O, for example) if glibc will not use those services. So there has been a search for alternatives, most of which involve creating a separate space for "system" file descriptors. Linus [suggested](<https://lwn.net/Articles/236844/>) one way of doing this: 

Which *could* be something as simple as saying "bit 30 in the file descriptor specifies a separate fd space" along with some flags to make open and friends return those separate fd's. That makes them useless for "select()" (which assumes a flat address space, of course), but would be useful for just about anything else. 

Davide Libenzi took this idea forward with [a patch to create a non-sequential file descriptor area](<http://lwn.net/Articles/236724/>). The current kernel tracks file descriptors in a linear array - a technique which works well as long as the "lowest available descriptor" rule holds. As soon as one starts setting high-order bits in file descriptor numbers, however, the linear array becomes rather less practical. So Davide's patch creates a separate, linked-list data structure used for the non-sequential file descriptor range. The [second part of the patch set](<http://lwn.net/Articles/236725/>) then fixes up the `dup2()` system call to use the new file descriptor range. The normal behavior of `dup2()` has not changed, but if the destination file descriptor is passed as `FD_UNSEQ_ALLOC`, a random file descriptor will be allocated from the non-sequential area. A specific file descriptor in that area can be requested by passing a number higher than `FD_UNSEQ_BASE`. 

This approach has the advantage of not requiring any new system calls or changing the default user-space binary interface at all. But [according to Ulrich Drepper](<https://lwn.net/Articles/236847/>), that attribute is not an advantage at all. Since using this capability requires application changes in any case, Ulrich would rather just see a new system call created; he proposes: 

```
int nonseqfd(int fd, int flags);
```

This system call would duplicate the open file descriptor `fd` into the non-sequential space, optionally closing `fd` in the process. The `flags` parameter would allow other attributes of the new file descriptor to be controlled. Of particular interest is whether that descriptor shows up in the `/proc/_pid_ /fd` directory. The optimal way of closing all open file descriptors, apparently, is to read that directory to see which descriptors are currently open. Keeping special descriptors out of that directory (perhaps shifting them to a parallel `private-fd` directory) will prevent well-meaning applications from closing the library's file descriptors. 

It has been suggested that the `open()` system call should get a flag which would cause it to select a non-sequential file descriptor from the outset, eliminating the need for a separate call to `nonseqfd()`. There are, however, a number of system calls which create file descriptors but which have no flags parameter and which, thus, will never be able to return non-sequential file descriptors; `socket()` is a classic example. So there will still be a need for a system call which can duplicate a file descriptor into the new space. 

Ulrich has requested that _all_ file descriptors in the non-sequential space be allocated randomly. He would rather not ever see a situation where application developers think they can rely on any specific allocation behavior when using that space. There have also been suggestions that the non-sequential space could be useful for for high-performance applications which hold large numbers of file descriptors open - web servers, for example. Such applications usually have no use for the "lowest available descriptor" guarantee and would happily do without the overhead of implementing that guarantee. Davide's current implementation does not appear to have been written with thousands of non-sequential file descriptors in mind, though. 

On another front, Ulrich has been working on a race condition which comes up with certain types of applications. It is possible to request that a file descriptor be automatically closed if the process performs an `exec()`; the `fcntl()` system call is used for this purpose. The problem is that there is some time between when the file descriptor is created (with an `open()` call, perhaps) and the subsequent `fcntl()` call. If another thread forks and runs a new program between those two calls, its copy of the new file descriptor will not have the close-on-exec flag set and will thus remain open. 

Solving that problem generally will take some work, but fixing the `open` case is relatively easy. Ulrich is [proposing](<http://lwn.net/Articles/236486/>) a new `O_CLOEXEC` flag for this purpose. There does not appear to be much opposition to this idea, so the new flag might well make an appearance in 2.6.23.
