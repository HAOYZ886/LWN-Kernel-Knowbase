---
title: Grabbing file descriptors with pidfd_getfd()
url: https://lwn.net/Articles/808997/
date: "January 9, 2020"
category: System calls
author: "By Jonathan Corbet January 9, 2020"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 9, 2020

In response to a growing desire for ways to control groups of processes from user space, the kernel has added a number of mechanisms that allow one process to operate on another. One piece that is currently missing, though, is the ability for a process to snatch a copy of an open file descriptor from another. That gap may soon be filled, though, if the [`pidfd_getfd()` system-call patch set](<https://lwn.net/ml/linux-kernel/20200107175927.4558-1-sargun@sargun.me/>) from Sargun Dhillon is merged. 

One thing that is possible in current kernels is to open a file that another process also has open; the information needed to do that is in each process's `/proc` directory. That does not work, though, for file descriptors referring to pipes, sockets, or other objects that do not appear in the filesystem hierarchy. Just as importantly, though, opening a new file in this way creates a new entry in the file table; it is _not_ the entry corresponding to the file descriptor in the process of interest. 

That distinction matters if the objective is to modify that particular file descriptor. One use case mentioned in the patch series is using seccomp to intercept attempts to bind a socket to a privileged port. A privileged supervisor process could, if it so chose, grab the file descriptor for that socket from the target process and actually perform the bind — something the target process would not have the privilege to do on its own. Since the grabbed file descriptor is essentially identical to the original, the bind operation will be visible to the target process as well. 

For the sufficiently determined, it is actually possible to extract a file descriptor from another process now. The technique involves using `[ptrace()](<http://man7.org/linux/man-pages/man2/ptrace.2.html>)` to attach to that process, stop it from executing, inject some code that opens a connection to the supervisor process and sends the file descriptor via an `SCM_RIGHTS` datagram, then running that code. This solution might justly be said to be slightly lacking in elegance. It also requires stopping the target process, which is likely to be unwelcome. 

This functionality, without the need to stop the target process, is relatively easy to implement in the kernel, though; a supervisor process would merely need to make a call to: 

```
int pidfd_getfd(int pidfd, int targetfd, unsigned int flags);
```

The target process is specified by `pidfd` (which is, as one might expect, a [pidfd](<https://lwn.net/Articles/794707/>), presumably obtained when the process was created). The file descriptor to grab is given by `targetfd`; if all goes well, the return value will be a local file-descriptor number corresponding to the target process's file. For all to go well, the calling process must have the ability to call `ptrace()` on the target process. 

The `flags` argument is currently unused and must be zero. There are, evidently, plans to add flags in the future, though. One would cause the file descriptor to be closed in the target process after being copied to the caller, thus truly "stealing" the descriptor from the target. Another would remove any related control-group data from socket file descriptors during the copy operation. 

This patch set has been through an impressive number of versions — and a fair amount of evolution — since it was first posted on December 5\. The [initial version](<https://lwn.net/ml/linux-kernel/20191205234450.GA26369%40ircssh-2.c.rugged-nimbus-611.internal//>) added a new `PTRACE_GETFD` command to `ptrace()`. [Version 3](<https://lwn.net/ml/linux-kernel/20191217005842.GA14379%40ircssh-2.c.rugged-nimbus-611.internal/>) switched to an `ioctl()` operation on a pidfd instead. In [version 5](<https://lwn.net/ml/linux-kernel/20191220232746.GA20215%40ircssh-2.c.rugged-nimbus-611.internal/>), fifteen days after the initial posting, this functionality moved into a separate system call. The current posting is version 9\. 

From the beginning there has not been much concern about the goals behind this feature; the comments have mostly focused on the implementation. At this point, Dhillon would appear to have just about exhausted the set of possible implementations — though some might be justified in thinking that a BPF version in the near future is inevitable. Failing that, this new system call may well be on track for the 5.6 or 5.7 merge window.
