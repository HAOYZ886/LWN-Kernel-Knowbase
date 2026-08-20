---
title: More fun with file descriptors
url: https://lwn.net/Articles/237722/
date: "June 12, 2007"
category: File descriptors; System calls
author: 
---

In [last week's episode](<http://lwn.net/Articles/236843/>), the kernel developers were considering the addition of a couple of flags to the `open()` system call; these flags would allow applications to select previously unavailable features like the non-sequential file descriptor range or immediate close-on-exec behavior. The problem that comes up quickly is that `open()` is just one of many system calls which creates file descriptors; most of the others do not have a parameter which allows an application to pass a set of accompanying flags. So it is not possible to request, for example, the non-sequential behavior when obtaining a file descriptor with `socket()`, `pipe()`, `epoll_create()`, `timerfd()`, `signalfd()`, `accept()`, and so on. 

In [the second version of the non-sequential file descriptor patch](<http://lwn.net/Articles/237301/>), Davide Libenzi attempted to address part of the problem by adding [a `socket2()` system call](<https://lwn.net/Articles/237727/>) with an added "flags" parameter. That was enough to frighten a number of developers; nobody really wants to see a big expansion of the system call list resulting from the addition of variations on all the file-descriptor-creating calls. Another approach, it seems, is required, but finding that approach is not entirely easy. 

One possibility is to simply ignore the problem; not everybody is sold on the need for non-sequential file descriptors or immediate close-on-exec behavior. There are enough people who see a problem here to motivate some sort of solution, though. Ulrich Drepper, the glibc maintainer, has seen enough applications to conclude that the issue is real. 

An alternative, [suggested](<https://lwn.net/Articles/237731/>) by Alan Cox, is to create a process state flag which controls the use of these features. So a call like: 

```
prctl(PR_SPARSEFD, 1);
```

would turn on non-sequential file descriptor allocation for all system calls made by the calling process. The problem here is that the lowest-available-descriptor behavior is a documented part of the POSIX binary interface. A process could waive that guarantee for itself, but it will always be hard to know that all libraries used by that process are safe in the absence of that behavior. One library might want to use non-sequential file descriptors, but that library cannot safely turn them on for the whole process without risking the creation of difficult bugs in obscure situations. It has been suggested that linker tricks could be used to avoid bringing older libraries, but Ulrich [feels](<https://lwn.net/Articles/237734/>) that people would respond by simply recompiling the older libraries and the potential bugs would remain. 

Linus [came into the discussion](<https://lwn.net/Articles/237737/>) with a statement that neither adding a bunch of new system calls nor the global flag were acceptable. Instead, he came up with a completely different idea: create a mechanism which allows a single system call to be invoked with a specific set of flags. His proposed interface is: 

```
int syscall_indirect(unsigned long flags, sigset_t sigmask,
                             int syscall, unsigned long args[6]);
```

The result would be a call to the given system call with the requested arguments. For the duration of the call, the given `flags` would be in effect, and signals in `sigmask` would be blocked. Even before adding any flags, this mechanism could be used to implement the series of system calls (`pselect()`, for example) which exists only to apply a signal mask to an earlier version of the call. Then the non-sequential file descriptor and close-on-exec behavior could be requested via the `flags` argument. Beyond that, flags could be added to control the handling of symbolic links, and various other things. Matt Mackall suggested that the "syslet" mechanism could be implemented as a "run this call asynchronously" flag. 

This approach is not without its potential problems. There are worries that the `flags` bits could be quickly exhausted, once again making it hard to add options to existing system calls. Linus [suggests](<https://lwn.net/Articles/237745/>) overloading the flag bits as a way of making them last longer. That approach risks problems if application developers attempt to apply the wrong flags for a given system call - there would be no automatic way of catching such errors - but it is unlikely that applications would be calling `syscall_indirect()` themselves, so this risk is relatively small. It is appropriate to worry about whether any conceivable, sensible behavior modification is covered by this interface, or whether it needs a different set of parameters. And one might well wonder whether, some years from now, a large percentage of system calls will be made via `syscall_indirect()`. 

This new system call suffers from one other shortcoming as well: there is currently no working implementation. That will likely change at some point, leading to a wider discussion of the proposed interface. If it still seems like a good idea, we might just have a way of adding new behavior to old functions without an explosion in the number of system calls. Sometimes, perhaps, it really is true that problems in computer science are best solved through the addition of another level of indirection.
