---
title: Completing the pidfd API
url: https://lwn.net/Articles/794707/
date: "July 26, 2019"
category: pidfd
author: "By Jonathan Corbet July 26, 2019"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 26, 2019

Over the last few kernel releases, the kernel has gained the concept of a "pidfd" — a file descriptor that represents a process. What started as a way of sending signals to processes without race conditions has evolved into a more complete process-management interface. Now one of the last pieces is being put into place: the ability to wait for processes using pidfds. But, naturally, that API has had to go through some revisions first. 

#### A pidfd recap

Unix-like systems traditionally represent many objects as files, but processes have always been an exception. They are, instead, represented by process IDs (PIDs), which are small integers — limited to 32767 by default, though that limit can be raised on Linux systems. There are a few problems with this representation, but the biggest one is arguably that PIDs are reused; when a process exits, its PID can be assigned to a new, unrelated process, and this can happen quickly. That creates a race condition where code that operates on a process, most often by sending it a signal, might end up performing an action on the wrong process. 

A pidfd is, instead, a file descriptor that refers to an existing process. Once the pidfd exists, it will only refer to that one process, so it can be used to send signals without worry that the wrong process might end up being the recipient. This feature is valuable enough that some process-management systems, most notably the one used by Android, are being rewritten to take advantage of it. 

There are two ways to create a pidfd. The preferred method in most cases will be to supply the `CLONE_PIDFD` flag to the [`clone()`](<http://man7.org/linux/man-pages/man2/clone.2.html>) system call (or perhaps [`clone3()`](<https://lwn.net/Articles/792628/>) in the future); upon successful process creation, a pidfd representing the child will be returned to the parent. It is also possible to create a pidfd for an existing process with [`pidfd_open()`](<https://lwn.net/Articles/789023/>), which was merged for the 5.3 kernel. 

A process holding a pidfd for a process can send a signal to that process using `pidfd_send_signal()`: 

```
int pidfd_send_signal(int pidfd, int signal, siginfo_t *info, unsigned int flags);
```

The 5.3 kernel also adds the ability to pass a pidfd to [`poll()`](<http://man7.org/linux/man-pages/man2/poll.2.html>), which will provide a notification when the process represented by that pidfd exits. 

#### Waiting on a pidfd

While it is now possible to use `poll()` to learn when a process has exited, that is not a complete solution for process-management systems, which need to be able to wait for specific processes and reap the exit information once they are done. That requires some sort of variant on the [`wait()`](<http://man7.org/linux/man-pages/man2/waitpid.2.html>) system call. To fill in that gap, Christian Brauner [proposed](<https://lwn.net/ml/linux-kernel/20190724144651.28272-1-christian@brauner.io/>) the addition of yet another new system call: 

```
int pidfd_wait(int pidfd, int *stat_addr, siginfo_t *info,
        		   struct rusage *rusage, int states, int flags);
```

This call would wait for the given `pidfd`; the `states` parameter can be used to specify which state transitions (`WSTOPPED` for when the process receives a stop signal, for example) to wait for. The `flags` field offers additional options, including `WNOHANG` for non-blocking operation; see the above-linked patch cover letter for the full list. 

This call, Brauner said, is ""one of the few missing pieces to make it possible to manage processes using only pidfds"". It is destined to remain missing, though, at least in that form; Linus Torvalds [made it clear](<https://lwn.net/ml/linux-kernel/CAHk-=whZPKzbPQftNGFB=iaSZGTSXNkhUASWF2V53MwB+A4zAQ@mail.gmail.com/>) that he didn't like it. He had no objection to the desired functionality, but questioned the need for a new system call; instead, he said, the `waitid()` system call should simply be extended with a new flag. 

That is exactly what was done in [a new patch series](<https://lwn.net/ml/linux-kernel/20190726093934.13557-1-christian@brauner.io/>) posted by Brauner; `waitid()` has gained a new `P_PIDFD` ID-type value that causes the given ID to be interpreted as a pidfd. This approach ended up being a rather smaller patch that does not need to add a new system call; there have been no responses to it as of this writing, but it would be unsurprising if this change were to be merged for 5.4. 

Beyond the ability to unambiguously specify which process should be waited for, this change will eventually enable another interesting feature: it will make it possible to wait for a process that is not a child — something that `waitid()` cannot do now. Since a pidfd is a file descriptor, it can be passed to another process via an `SCM_RIGHTS` datagram in the usual manner. The recipient of a pidfd will, once this functionality is completed, be able to use it in most of the ways that the parent can to operate on (or wait for) the associated process. 

There was one other interesting piece in the original `pidfd_wait()` proposal: [a new `clone()` flag (`CLONE_WAIT_PID`)](<https://lwn.net/ml/linux-kernel/20190724144651.28272-5-christian@brauner.io/>) that would cause the newly created process to be invisible to most `wait()` calls. Only a variant of `wait()` that specified that process in particular (by specifying its pidfd, for example) would be able to reap its exit information. There are a few use cases for this functionality; one that was listed is a library that needs to create a helper process that won't show up if the calling application calls `wait()`. This feature was not part of the second patch set, but is expected to show up in a separate posting in the near future. 

There will almost certainly be other pidfd-oriented enhancements in the future; this feature is new and should not be considered to be complete. But the ability to wait on a pidfd might be seen as the end of the first round of development for the pidfd concept. It has been a relatively quiet set of changes, but the move to pidfds is a fundamental change in how processes are managed on Linux systems.
