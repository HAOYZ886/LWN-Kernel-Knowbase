---
title: "Kernel-based checkpoint and restart"
url: https://lwn.net/Articles/293575/
date: "August 11, 2008"
category: Checkpointing; Containers
author: "By Jonathan Corbet August 11, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 11, 2008

Your editor, who has carefully hidden several years of experience in Fortran-based scientific programming from this readership, encountered checkpoint and restart facilities a long time ago. In those days, programs which would run for days of hard-won CPU time on an unimaginably fast CDC or Cray mainframe would occasionally checkpoint themselves, minimizing the amount of compute time lost when (not if) the system went down at an inopportune time. It was a sort of insurance policy, with the premiums being paid in the form of regular checkpoint calls. 

Central processor time is no longer in such short supply, but there is still interest in the ability to checkpoint a running application and restore its state at some future time. One obvious application of this capability is to restore the application on a different machine; in this way, running applications can be moved from one host to another. If the "application" is an entire container full of tasks, you now have the ability to shift those containers around without the contained tasks even being aware of what is going on. That, in turn, can provide for load balancing, or just the ability to move containers off a machine which is being taken down. 

Linux does not have this capability now. Anybody who thinks about adding it must certainly find the prospect daunting; applications have a _lot_ of state hidden throughout the system. This state includes open files (and positions within the files), network sockets and pipes connected to remote peers, signal states, outstanding timers, special-purpose file descriptors (for `epoll_wait()`, for example), `ptrace()` status, CPU affinities, SYSV semaphores, futexes, SELinux state, and much more. Any failure to save and properly restore all of that state will result in a broken process. It is no wonder that Linux does not do checkpoint and restart; most rational developers would be driven away by the complexities involved in making it work in an even remotely robust manner. 

But, then, there was a time when rational programmers would not have attempted the creation of Linux in the first place. So it should not be surprising to see that developers are working on the checkpoint and restart problem. The latest attempt can be seen in [this patch set](<http://lwn.net/Articles/293533/>) posted by Dave Hansen (but originally written by Oren Laadan). It is far from being ready for prime-time use, but it does show the sort of approach which is being taken. 

For some time, the prevailing wisdom was that checkpoint and restart should be pushed as much into user space as possible. A user-space process could handle the marshaling of process state and writing it to a file; the kernel would only get involved when it was strictly necessary. It turns out, though, that this involvement is required fairly often, requiring the addition of "lots of new, little kernel interfaces" to make everything work. So, at a meeting at OLS, the checkpoint/restart developers decided to take a different approach and move the work into the kernel. The result is the creation of just two new system calls: 

```
int checkpoint(pid_t pid, int fd, unsigned long flags);
        int restart(int crid, int fd, unsigned long flags);
```

A call to `checkpoint()` will write an image of the current process to the given `fd`. The `pid` argument identifies the init process for the current process's container; it is saved to the image but not otherwise used in the current patch. If the operation succeeds, the return value will be a unique (until the system reboots) "checkpoint image identifier". `restart()` reverses the process; `crid` is the image identifier, which is not currently used. The `flags` argument is currently unused in both system calls. These interfaces seem likely to change; future enhancements to the interface are likely to include capabilities like checkpointing other processes and groups of processes. 

The `CAP_SYS_ADMIN` capability is currently required for both `checkpoint()` and `restart()`. That is somewhat unfortunate, in that it would be nice if ordinary, unprivileged processes were able to checkpoint and restart themselves. There are some real security implications which must be kept in mind, though, especially when one considers the sort of damage that could result from an attempt to restart a carefully-manipulated checkpoint image. Making `restart()` secure for unprivileged use will not be a job for the faint of heart. 

At this stage of development, the patch does not even attempt to solve the entire problem. It is able to save the current state of virtual memory (but only in the absence of non-private, shared mappings), current processor state, and the contents of the task structure. That is enough to checkpoint and restart a "hello, world" program, but not a whole lot more. But that is a reasonable place to start. Given the complexity of the problem, proceeding in careful baby steps seems like the right way to go. So we're probably not going to have a working checkpoint facility in the kernel in the near future, but, with luck and patience, we'll eventually have something that works.
