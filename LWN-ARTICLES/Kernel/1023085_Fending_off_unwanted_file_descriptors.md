---
title: Fending off unwanted file descriptors
url: https://lwn.net/Articles/1023085/
date: "June 5, 2025"
category: "Networking-SCM RIGHTS; Releases-6.16"
author: "By Jonathan Corbet June 5, 2025"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 5, 2025

One of the more obscure features provided by Unix-domain sockets is the ability to pass a file descriptor from one process to another. This feature is often used to provide access to a specific file or network connection to a process running in a relatively unprivileged context. But what if the recipient doesn't _want_ a new file descriptor? A feature added for the 6.16 release makes it possible to refuse that offer. 

Normally, a Unix-domain connection is established between two processes to allow the exchange of data. There is, however, a special option (`SCM_RIGHTS`, documented in [unix(7)](<https://man7.org/linux/man-pages/man7/unix.7.html>)) to the [`sendmsg()`](<https://man7.org/linux/man-pages/man2/sendmsg.2.html>) system call that accepts a file descriptor as input. That descriptor will be duplicated and installed into the receiving process, giving the recipient access to the file as if it had opened it directly. `SCM_RIGHTS` messages can be used to give a process access to files that would otherwise be unavailable to it. It is also useful for network-service dispatchers, which can hand off incoming connections to worker processes. 

The `SCM_RIGHTS` feature is not exactly new; it was added to the 1.3.71 development kernel by Alan Cox in 1996, but existed in Unix prior to that. Interestingly, it seems that, in the long history of this feature, nobody has ever considered the question of whether the recipient actually wants to acquire a new file descriptor. In retrospect, it seems like a bit of a strange omission. Developers tend to take care with the management of the open-file table in their programs, closing files that are no longer needed, and ensuring that file descriptors are not passed into new process or programs unnecessarily. Injecting an unexpected file descriptor into a process has the potential to interfere with those efforts. 

A specific problem with unexpected file descriptors, as [pointed out](<https://lwn.net/ml/all/20250519205820.66184-9-kuniyu@amazon.com>) by Kuniyuki Iwashima in [this patch series](<https://lwn.net/ml/all/20250519205820.66184-1-kuniyu@amazon.com>), is their denial-of-service potential. If a file descriptor that is somehow hung — consider a descriptor for an attacker-controlled FUSE filesystem or a hung NFS file — is installed into a process, the recipient may be blocked indefinitely while trying to close it. This situation is similar to dumping a load of toxic waste on somebody's lawn; the victim may find themselves unable to get rid of it. In the `SCM_RIGHTS` case, this sort of toxic file descriptor can prevent the recipient from getting work done (or exiting). 

The solution, as implemented by Iwashima, is to provide a new option to disable the reception of file descriptors over a given socket. That is done with a [`setsockopt()`](<https://man7.org/linux/man-pages/man3/setsockopt.3p.html>) call, using the new `SO_PASSRIGHTS` flag, like: 

```
int zero = 0;
        ret = setsockopt(fd, SOL_SOCKET, SO_PASSRIGHTS, &zero, sizeof(zero));
```

If this option is used as above to disable the reception of file descriptors, any attempt to transfer a descriptor over that socket will fail with an `EPERM` error. Of course, the reception of `SCM_RIGHTS` file descriptors remains enabled by default; to do otherwise would surely break large numbers of programs. If `SCM_RIGHTS` were being designed today, it would likely require an explicit opt-in, but that ship sailed decades ago, so developers wanting to protect a process against unwanted file descriptors will need to disable `SCM_RIGHTS` explicitly for any socket that it might be pass to [`recvmsg()`](<https://man7.org/linux/man-pages/man3/recvmsg.3p.html>). 

The `SO_PASSRIGHTS` option found its way into the mainline kernel (as part of [the large networking pull](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=1b98f357dadd>)) on May 28 and will be available as of the 6.16 kernel release.
