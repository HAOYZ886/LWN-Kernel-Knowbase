---
title: Optional mandatory locking
url: https://lwn.net/Articles/667210/
date: "December 9, 2015"
category: "Filesystems-POSIX locks"
author: "By Jonathan Corbet December 9, 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 9, 2015

Files in a filesystem are globally visible data structures, so it is entirely possible that more than one process will try to modify the same file at the same time. In the absence of a way to coordinate the actions of those processes, the result will almost certainly be messy at best. One form of coordination is mandatory locking, which has been supported by Linux since nearly the beginning. A recent discussion, however, may be the beginning of the end for this venerable, if unloved, feature. 

Back in the early days, Unix lacked a mechanism for serializing access to shared files. Eventually, POSIX standardized an advisory locking mechanism invoked via the `fcntl()` system call; with advisory locks, a process can lock a region of a file for reading or writing, the latter granting exclusive access to that region. POSIX advisory locking mostly works, modulo [certain shortcomings](<http://0pointer.de/blog/projects/locking.html>), and it was recently improved in Linux with the addition of [file-private POSIX locks](<https://lwn.net/Articles/586904/>). But there is one obvious problem with advisory locks: they're advisory. All it takes is one process that isn't playing by the rules and the whole scheme falls down. As has been noted over the years, there is a lot of poorly written software out there; while advisory locks can work well within a single, well-written application (think of a database manager accessing its files, for example), they cannot be relied on in situations where arbitrary programs may try to access a file. 

The answer from POSIX was mandatory locks, which can be thought of as advisory locks with more kludges added on top. If a file is made subject to mandatory locks (see below), then the locks that would have otherwise been advisory are enforced by the kernel. If a process write-locks a region of a file, any other process trying to read or write that region will be blocked until the lock is released. With mandatory locks, there is no longer a need to hope that every process accessing a file will observe the advisory locks. 

Enforcing mandatory locks on every file access would be expensive, so the mandatory locks mechanism is restricted in scope. First of all, the filesystem must be mounted with the "`-o mand`" option to enable mandatory locks. Then any file subject to mandatory locking must be marked by setting the set-group-ID protection bit (but not the group execute bit). When those conditions are met, mandatory locks will be enforced on the file in question — to a point. 

Linux has supported mandatory locks for a long time; the [document describing them](<https://lwn.net/Articles/667215/>) was written in 1996. It compares the kernel's implementation to the equivalent in other bleeding-edge operating systems like SunOS 4.1, Solaris 2, and HP-UX 9\. Linux comes off relatively well in that comparison, but that is setting a low bar; everybody's implementation of mandatory locks was evidently bug-ridden, inconsistent, and unreliable. The kernel's document itself starts off with a section (added in 2007) titled "why you should avoid mandatory locking." Among other things, in Linux, the lock restrictions are enforced only at the beginning of an operation, so operations can race with locks that are established (by another process) halfway through. 

In other words, Linux claims to support mandatory locks, but that support is incomplete and racy; developers who are serious about getting user-space locking right understand this and use something else. 

While working on user namespaces, Eric Biederman recently [ran into another issue](<https://lwn.net/Articles/667216/>): a process within a namespace can apply a mandatory lock to a file, then pass a descriptor for that file to a daemon outside of the namespace. That daemon will then freeze as soon as it tries to access the file descriptor. This is the sort of denial-of-service attack that namespaces are supposed to prevent, so this behavior is a bit of a problem. It can be fixed easily enough by limiting mandatory-lock enforcement to other processes in the same namespace, but Eric also has a more far-reaching solution in mind. 

As has been noted, mandatory locks are inelegant, buggy, and subject to races on multiple operating systems. Furthermore, they have been that way for decades, and nobody has made the effort to fix them. Rather than try to fix the problems at this late date, Eric [suggested](<https://lwn.net/Articles/667217/>): 

From what little I can glean we want to discourage people from using mandatory locking and to let it wither and die. A Kconfig option that allows mandatory locking to be disabled at compile time seems like the first step in making that happen. Perhaps in a decade or so when all linux distributions are setting the option we can remove the code. 

The thinking here is that mandatory locking probably has almost no users at all. That might argue for its immediate removal, but that "almost" is the sticking point: breaking even a single ancient application goes against the kernel's "no regressions" rule. So the next best thing is to slowly make the feature harder to get at and to see if anything breaks. 

The resulting discussion among filesystem developers was remarkably one-sided; there is little love for the mandatory-locking feature in the development community. There were some worries that Samba might rely on mandatory locking, but Jeremy Allison [put those concerns to rest](<https://lwn.net/Articles/667218/>). So Jeff Layton quickly [queued the patch](<https://lwn.net/Articles/667219/>) and said that, in the absence of objections, he would push it during the 4.5 merge window. No such objections have been heard, so it appears that, in 4.5, the mandatory locks feature will be optional and slated for eventual (if distant) removal.
