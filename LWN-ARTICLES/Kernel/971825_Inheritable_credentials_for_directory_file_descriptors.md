---
title: Inheritable credentials for directory file descriptors
url: https://lwn.net/Articles/971825/
date: "May 2, 2024"
category: "System calls-openat2"
author: "By Jonathan Corbet May 2, 2024"
---

By **Jonathan Corbet**  
May 2, 2024

In Unix-like systems, an open file descriptor carries the right to access the opened object in specific ways. As a general rule, that file descriptor does _not_ enable access to any other objects. The recently merged [BPF token feature](<https://lwn.net/Articles/947173/>) runs counter to this practice by creating file descriptors that carry specific BPF-related access rights. A similar but different approach to capability-carrying file descriptors, in the form of directory file descriptors that include their own credentials, is currently under consideration in the kernel community. 

Linux systems allow a process to open a directory with any of the variants of the [`open()` system call](<https://man7.org/linux/man-pages/man2/open.2.html>). The resulting "directory file descriptor" can be used to read the contents of the directory; it is also useful, when passed to system calls like `openat()`, to specify the starting directory for the pathname lookup of the file to be opened. A privileged process can open a directory and give the file descriptor to a less-privileged process (or simply drop its own privileges), and that descriptor will continue to be usable to access the directory, even if the owning process would otherwise be unable to do so. 

That access does not, however, extend to any files contained within that directory. 

Stas Sergeev recently [proposed](<https://lwn.net/ml/linux-kernel/20240423104002.9012-1-stsp2@yandex.ru/>) a change to that situation in the form of a new flag (`OA2_INHERIT_CRED`) for the [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>) system call. If a process uses that flag while opening a file, and that process provides a directory file descriptor, the file will be opened using the credentials that were in effect when the directory was opened. So, if a privileged process created the directory file descriptor, any other process owning that descriptor could open files in the reference directory using the privileged process's user and group IDs. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

In other words, when this flag is used, a directory file descriptor grants more than just access to the directory itself; it also provides credentials to access files within the directory. This feature can be used, according to Sergeev, to implement a sort of lightweight sandboxing mechanism to restrict a process (or a container) to a specific directory tree. Such restrictions can be implemented now, but is rather more cumbersome to set up. 

Andy Lutomirski [said](<https://lwn.net/ml/linux-kernel/4D2A1543-273F-417F-921B-E9F994FBF2E8@amacapital.net/>) that he liked the idea; ""it's a sort of move toward a capability system"". He added, though, that turning a directory file descriptor into this sort of capability should require an explicit act — it should not just happen by default. Not every process providing a directory file descriptor to another will want to hand over its rights to access objects in the directory as well. He also worried about potential mischief resulting from directory file descriptors opened in special filesystems like `/proc`. 

As a result of these comments, a number of changes had been made by the time that the patch series got to [version 6](<https://lwn.net/ml/linux-kernel/20240427112451.1609471-1-stsp2@yandex.ru/>). To be usable with the (renamed) `OA2_CRED_INHERIT` flag, a directory file descriptor must have been opened with the new `O_CRED_ALLOW` flag. An attempt to use the `OA2_CRED_INHERIT` flag on a directory file descriptor created without `O_CRED_ALLOW` will just result in an `EPERM` error. The kernel will also reject `OA2_CRED_INHERIT` opens that involve `/proc` or symbolic links that lead out of the directory. Any file descriptors opened using `OA2_CRED_INHERIT` will be automatically closed in an [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>) call. 

Meanwhile, `O_CRED_ALLOW` directory file descriptors cannot be passed to any other process over a Unix-domain socket. This would appear to be the only case where the `SCM_RIGHTS` mechanism restricts the type of file descriptor that can be passed in this way. This restriction prevents a container from giving its special permissions to a process outside of the container, but it will also block attempts to pass an `O_CRED_ALLOW` file descriptor _into_ a container. For the intended use case (where a privileged process sets up the file descriptor before dropping privileges) this restriction will not be a problem, but it could possibly impede other use cases. 

Sergeev [notes](<https://lwn.net/ml/linux-kernel/20240427112451.1609471-4-stsp2@yandex.ru/>) in the series that, if this idea is accepted, there are more patches to come: 

> This patch is just a first step to such sandboxing. If things go well, in the future the same extension can be added to more syscalls. These should include at least unlinkat(), renameat2() and the not-yet-upstreamed setxattrat(). 

Whether things will, in fact, go well is yet to be determined; this sort of security-related change to a core system call tends to need a high degree of review. And, of course, there will be people with other ideas of how this functionality could be provided. For example, Lutomirski [proposed](<https://lwn.net/ml/linux-kernel/CALCETrUL3zXAX94CpcQYwj1omwO+=-1Li+J7Bw2kpAw4d7nsyw@mail.gmail.com/>) a somewhat more elaborate mechanism where credentials could be attached using `open_tree()` (which is part of [the new(ish) mount API](<https://lwn.net/Articles/759499/>)); a process could then mount the given subtree as a separate filesystem. This would allow him to ""pick a host directory, pick a host *principal* (UID, GID, label, etc), and have the *entire container* access the directory as that principal"". 

Lutomirski was seeking comments on this approach and did not include an implementation of this idea. [The comment he got](<https://lwn.net/ml/linux-kernel/20240429-donnerstag-behilflich-a083311d8e00@brauner/>) came from filesystem-layer maintainer Christian Brauner, who pointed out that [ID-mapped mounts](<https://lwn.net/Articles/896255/>) can already provide most of the functionality that Lutomirski appeared to be looking for. Lutomirski has not yet responded to indicate whether he agrees. 

It may take some time to see whether this work is accepted, and in which form. Adding new security features to an operating-system kernel needs to be done with care; there can often be surprising interactions with existing features, and they may be used in surprising ways. [Serious vulnerabilities](<https://lwn.net/Articles/961086/>) have resulted from file descriptors passed into containers in the recent past; developers would want to be sure that this feature would not lead to similar problems. But, regardless of how this specific patch set is ultimately received, it does demonstrate a direction — toward more capability-oriented systems — that many developers would like to pursue.
