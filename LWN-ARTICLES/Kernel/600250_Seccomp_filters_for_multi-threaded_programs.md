---
title: "Seccomp filters for multi-threaded programs"
url: https://lwn.net/Articles/600250/
date: "May 29, 2014"
category: "Security-seccomp"
author: "By Jonathan Corbet May 29, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 29, 2014

The secure computing ("seccomp") mechanism helps in the sandboxing of processes by restricting access to system calls. Seccomp works by attaching one or more programs to a process; those programs, written in Berkeley packet filter (BPF) byte code, are invoked for every system call made by the affected process. The BPF filter programs have access to the system call number and arguments; each filter has the option of denying the system call. Seccomp filters can thus restrict access to specific system calls, or, for example, only allow `write()` to be called on specific file descriptors. This mechanism works well as far as it goes, but it was not designed for use with multi-threaded programs. A set of proposed changes should close that particular functionality gap in the near future, though. 

In current kernels, a process can apply a filter program to itself with the `prctl()` system call: 

```
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, filter);
```

Where `filter` is a pointer to a `sock_fprog` structure containing the BPF program to be applied. Multiple programs can be added with multiple `prctl()` calls; each will be executed in sequence and any can reject a system call. There is no mechanism for removing filters once they have been applied to a process. Adding filters is normally a privileged operation; otherwise there is a real risk of privilege escalation via setuid programs that do not expect some operations to be denied (see [this old sendmail vulnerability](<https://sites.google.com/site/fullycapable/Home/thesendmailcapabilitiesissue>) for an example). But any process may set filters on itself if it has first called: 

```
prctl(PR_SET_NO_NEW_PRIVS, 1);
```

to disable the addition of any privileges to the process. In particular, a process marked as "no new privileges" cannot gain capabilities or access to a different user ID by running setuid or setgid programs. 

What is lacking in the current interface is any way for a process to apply filters to a different process (or thread). There does not seem to be a use case for the ability to add filters to arbitrary processes; among other things, trying to contain a program that is already off and running would be a recipe for unpleasant race conditions. But it seems that there is value in allowing a thread to apply filters to its sibling threads. In the absence of this ability, it can be hard to ensure that a seccomp filter applies to _all_ threads running as part of a process. Threads inherit their parent's filters when they are created, but any threads created before the filters are applied will remain uncontained. It may not always be practical to set up the filters before any threads are created, so the ability to attach them to threads after creation is a useful way to ensure that no part of a program escapes filtering. 

Adding that ability is the object of [this patch set](<https://lwn.net/Articles/600103/>) from Kees Cook. All Kees really needed to do was to add an "apply this filter to all threads" flag to the `PR_SET_SECCOMP` operation, but, [as so often seems to be the case](<https://lwn.net/Articles/585415/>), that operation was defined without the ability to pass in additional flags to modify its behavior. So, instead, Kees has added a new operation: 

```
prctl(PR_SECCOMP_EXT, SECCOMP_EXT_ACT, SECCOMP_EXT_ACT_FILTER, flags, filter);
```

If `flags` is zero, this operation behaves just like the `PR_SET_SECCOMP` example above; it attaches `filter` to the calling process. But if the `SECCOMP_FILTER_TSYNC` flag is set, the given `filter` (along with any other filters already applied to the calling process) will be applied to all threads in the process's thread group, thus ensuring that all threads are running with the same set of filters. 

There is one other new operation: 

```
prctl(PR_SECCOMP_EXT, SECCOMP_EXT_ACT, SECCOMP_EXT_ACT_TSYNC, 0, 0);
```

This one will apply the calling process's filters to all other threads without making any changes to the filters themselves. 

In either case, other threads will only have their filtering changed if whatever filter they currently have applied is an "ancestor" of the filters running on the calling process. Essentially, any filters applied to the target thread must also have been applied to the calling thread; any thread that has a totally unrelated filter will not have its filtering changed. If a thread is not running with a filter at all, it will be put into the seccomp mode and the filters will be applied. Also, if the calling thread has the "no new privileges" mode set, that mode will be set on all other threads as well. 

This is the fifth version of this patch set; the previous attempts needed work in response to locking and other issues. Unless another problem turns up, this code should be about ready for merging. There does not appear to be any opposition to the concept, so this feature could find its way into the mainline as early as 3.16.
