---
title: "Domesticating applications, OpenBSD style"
url: https://lwn.net/Articles/651700/
date: "July 21, 2015"
category: Security
author: "By Jonathan Corbet July 21, 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 21, 2015

One of the many approaches to improving system security consists of reducing the attack surface of a given program by restricting the range of system calls available to it. If an application has no need for access to the network, say, then removing its ability to use the `socket()` system call should cause no loss in functionality while reducing the scope of the mischief that can be made should that application be compromised. In the Linux world, this kind of sandboxing can be done using a security module or the `seccomp()` system call. OpenBSD has lacked this capability so far, but it may soon gain it via a somewhat different approach than has been seen in Linux. 

It is fair to characterize the sandboxing features in Linux as being relatively complex. The complexity of the security module options, and SELinux in particular, is legendary. The `seccomp()` system call has two modes: very simple (in which case almost nothing but `read()` and `write()` is allowed), or rather complex (a program written in the Berkeley packet filter (BPF) language makes decisions on system call availability). There is a great deal of flexibility available with both security modules and `seccomp()`, but it comes at a cost. 

OpenBSD leader Theo de Raadt is [particularly scornful](<https://lwn.net/Articles/651701/>) of the BPF-based approach: 

Some BPF-style approaches have showed up. So you need to write a program to observe your program, to keep things secure? That is insane. 

His posting contains a work-in-progress implementation of a simpler approach to sandboxing (mostly written by Nicholas Marriott, it seems) in the form of a system call named `tame()`. 

The core idea behind `tame()` is that most applications run in two phases: initialization and steady-state execution. The initialization phase typically involves opening files, establishing network connections, and more; after initialization is complete, the program may not need to do any of those things. So there is often an opportunity to reduce an application's privilege level as it moves out of the initialization phase. `tame()` performs that privilege reduction; it is thus meant to be placed within an application, rather than (as with SELinux) imposed on it from the outside. 

The system call itself is simple enough: 

```
int tame(int flags);
```

If `flags` is passed as zero, the only system call available to the process thereafter will be `_exit()`. This mode is thus suitable for a process cranking on data stored in shared memory, but not much else. For most real-world applications, the reduction in privilege will need to be a bit less heavy-handed. That is what the flags are for. If any flags at all are present, a base set of system calls, with read-only functionality like `getpid()`, is available. For additional privilege, specific flags must be used: 

  * `TAME_MALLOC` provides access to memory-management calls like `mmap()`, `mprotect()`, and more. 

  * `TAME_RW` allows I/O on existing file descriptors, enabling calls like `read()`, `write()`, `poll()`, `fcntl()`, `sendmsg()` and, interestingly, `pipe()`. 

  * `TAME_RPATH` enables system calls that perform pathname lookup without changing the filesystem: `chdir()`, `openat()` (read-only), `fstat()`, etc. 

  * `TAME_WPATH` allows changes to the filesystem: `chmod()`, `openat()` for writing, `chown()`, etc. Note that `TAME_RPATH` and `TAME_WPATH` both implicitly set `TAME_RW` as well. 

  * `TAME_CPATH` allows the creation and removal of files and directories via `rename()`, `rmdir()`, `link()`, `unlink()`, `mkdir()`, etc. 

  * `TAME_TMPPATH` enables a number of filesystem-related system calls, but only when applied to files underneath `/tmp`. 

  * `TAME_INET` allows `socket()` and related calls needed to function as an Internet client or server. 

  * `TAME_UNIX` allows networking-related system calls restricted to Unix-domain sockets. 

  * `TAME_DNSPATH` is meant to allow hostname lookups; it gives access to a few system calls like `socket()`, but only after the program successfully opens `/etc/resolv.conf`. So the kernel has to track whether a few "special" files like `resolv.conf` have been opened during the lifetime of the tamed process. 

  * `TAME_GETPW` enables the read-only opening of a few specific files needed for `getpwnam()` and related functions. It will also turn on `TAME_INET` if the program succeeds in opening `/var/run/ypbind.lock`. 

  * `TAME_CMSG` allows file descriptors to be passed with `sendmsg()` and `recvmsg()`. 

  * `TAME_IOCTL` turns on a few specific, terminal-related `ioctl()` commands. 

  * `TAME_PROC` allows access to `fork()`, `kill()`, and related process-management system calls. 

A process may make multiple calls to `tame()`, but it can only restrict its current capabilities. Once a particular flag has been cleared, it cannot be set again. 

The patch includes changes to a number of OpenBSD utilities. The `cat` command is restricted to `TAME_MALLOC` and `TAME_RPATH`, for example; never again will `cat` be able to run amok on the net. The `ping` command gets access to the net, instead, but loses the ability to access the filesystem. And so on. 

This system call has a number of features that may look a bit strange to developers used to Linux. It encodes quite a bit of policy in the kernel, including where the password database is stored and the use of [Yellow Pages/NIS](<https://en.wikipedia.org/wiki/Network_Information_Service>); one would grep in vain for `ypbind.lock` in the Linux kernel source. `tame()` may seem limited in the range of restrictions that it can apply to a process; it will almost certainly allow more than what is strictly needed in most cases. It thus lacks the flexibility that Linux developers typically like to see. 

On the other hand, using `tame()`, it was evidently possible to add restrictions to a fair number of system commands with a relatively small amount of work and little code. Writing _ad hoc_ BPF programs or SELinux policies to accomplish the same thing would have taken quite a bit longer and would have been more error-prone. `tame()`, thus, looks like a way to add another layer of defense to a program in a quick and standardized way; as such, it may, in the end, be used more than something like `seccomp()`. 

If the `tame()` interface proves to be successful in the BSD world, there is an interesting possibility on the Linux side: it should be possible to completely implement that functionality in user space using the `seccomp()` feature (though it would probably be necessary to merge one of the [patches](<https://lwn.net/Articles/590436/>) adding extended BPF functionality to `seccomp()`). We would then have the simple interface for situations where it is adequate while still being able to write more flexible filter policies where they are indicated. It could be the best of both worlds. 

The first step, though, would probably be to let the OpenBSD project explore this space and see what kind of results it gets. The ability to try out different models is one of the strengths that comes from having competing kernels out there. The ability to quickly copy that work is, instead, an advantage that comes from free software. If this approach to attack-surface reduction works out, we in the Linux world may, too, be able to `tame()` our `cat` in the future.
