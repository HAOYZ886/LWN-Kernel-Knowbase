---
title: "New system calls: pidfd_open() and close_range()"
url: https://lwn.net/Articles/789023/
date: "May 23, 2019"
category: System calls; pidfd
author: "By Jonathan Corbet May 23, 2019"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 23, 2019

The linux-kernel mailing list has recently seen more than the usual amount of traffic proposing new system calls. LWN is endeavoring to catch up with that stream, starting with a couple of proposals for the management of file descriptors. `pidfd_open()` is a new way to create a "pidfd" file descriptor that refers to a process in the system, while `close_range()` is an efficient way to close many open descriptors with a single call. 

#### pidfd_open()

There has been [a fair amount of development activity](<https://lwn.net/Articles/784831/>) around pidfds, which can be used to send signals to processes without worries that the target process may die and be replaced by another one using the same process ID. The 5.2 merge window saw the addition of a new `CLONE_PIDFD` flag to the [`clone()`](<http://man7.org/linux/man-pages/man2/clone.2.html>) system call. If that flag is present, the kernel will return a pidfd (referring to the newly created child) to the parent by way of the `ptid` argument; that pidfd can then be used to send signals to the child process at some future point. 

There are times, though, when it is not possible to create a process in this manner, but a management process would still like to get a pidfd for another process. Opening the target's `/proc` directory could work; that was once the only way to get a pidfd for a process. But the `/proc` approach is apparently not usable in all situations. On some systems, `/proc` may not exist (or be accessible) at all. For situations like this, Christian Brauner has [brought back an earlier proposal](<https://lwn.net/ml/linux-kernel/20190520155630.21684-1-christian@brauner.io/>) for a new system call to create a pidfd: 

```
int pidfd_open(pid_t pid, unsigned int flags);
```

The target process is identified with `pid`; the `flags` argument must be zero in the current proposal. The return value will be a pidfd corresponding to `pid`. It's worth noting that there is a possible race window here; `pid` could be recycled before `pidfd_open()` runs. That window is small in most normal usage, though, and there are ways for the caller to check and ensure that the process of interest is still running. 

When `pidfd_open()` was proposed in the past, it would return a different flavor of pidfd than would be obtained by opening `/proc`; an `ioctl()` call was provided to convert between the two. This behavior was not particularly popular, and has been dropped; there is now just a single type of pidfd, regardless of where it has been obtained. 

The lack of `pidfd_open()` is, Brauner says, the main obstacle keeping applications like Android's low-memory killer and systemd from using pidfds for process management. Once that has been resolved, ""they intend to switch to this API for process supervision/management as soon as possible"". Comments on this system call have settled down to relatively small implementation details, so it seems likely to go in during the 5.3 merge window. 

#### close_range()

One possibly surprising `pidfd_open()` feature is that the pidfd it creates has the `O_CLOEXEC` flag set automatically; that will cause the descriptor to be automatically closed should the owning process call [`execve()`](<http://man7.org/linux/man-pages/man2/execve.2.html>) to run a new program. This is a hardening feature, intended to prevent open file descriptors from leaking into places where they were not intended to be. David Howells has recently [proposed](<https://lwn.net/ml/linux-kernel/155800752418.4037.9567789434648701032.stgit@warthog.procyon.org.uk/>) changing [the new filesystem mounting API](<https://lwn.net/Articles/759499/>) to unconditionally set that flag as well. 

This change evoked [a protest](<https://lwn.net/ml/linux-kernel/20190516162259.GB17978@ZenIV.linux.org.uk/>) from Al Viro, who does not feel that changing longstanding Unix conventions is the right approach, especially since the behavior of existing calls like `open()` cannot possibly change in this way. He later [suggested](<https://lwn.net/ml/linux-kernel/20190516165021.GD17978@ZenIV.linux.org.uk/>) that a `close_range()` system call might be a better way to ensure that file descriptors are closed before calls like `execve()`. Brauner duly [implemented this idea](<https://lwn.net/ml/linux-kernel/20190521113448.20654-1-christian@brauner.io/>) for consideration. The new system call would be: 

```
int close_range(unsigned int first, unsigned int last).
```

A call to `close_range()` will close every open file descriptor from `first` through `last`, inclusive. Passing a number like `MAXINT` for `last` will work and is the expected usage much of the time. Closing descriptors in the kernel this way, rather than in a loop in user space, allows for a significant speedup; as Brauner put it, ""the performance is striking"", even though there are clearly ways in which the implementation could be made faster yet. 

This API is rather less settled at this point. Howells [suggested](<https://lwn.net/ml/linux-kernel/28114.1558456227@warthog.procyon.org.uk/>) something more like: 

```
int close_from(unsigned int first);
```

This variant would close all open descriptors starting with `first`. It seems that there are use cases, though, for leaving some high-numbered file descriptors open, so this version would be less useful. Florian Weimer, instead, [suggested](<https://lwn.net/ml/linux-kernel/87tvdoau12.fsf@oldenburg2.str.redhat.com/>) looking at [the Solaris `closefrom()` and `fdwalk()` functions](<https://docs.oracle.com/cd/E88353_01/html/E37843/closefrom-3c.html>) for inspiration. `closefrom()` is equivalent to Howells's `close_from()`, while `fdwalk()` allows a process to iterate through its open file descriptors. Weimer said that if the kernel were to implement a `nextfd()` system call to obtain the next open file descriptor, both `closefrom()` and `fdwalk()` could be implemented in the C library. 

The value of these functions [was not clear](<https://lwn.net/ml/linux-kernel/20190521130438.q3u4wvve7p6md6cm@brauner.io/>) to Brauner, though. In particular, `fdwalk()` appears to be mostly needed on systems that lack information on open file descriptors in `/proc`. In the absence of a pressing need for `nextfd()`, it is unlikely to be implemented, much less merged. So, unless some other proposal comes along and proves more interesting, a future `close_range()` implementation appears to be the most likely to find its way into a mainline kernel release.
