---
title: Making EPERM friendlier
url: https://lwn.net/Articles/532771/
date: "January 19, 2013"
category: "User-space API-Error reporting"
author: "By Michael Kerrisk January 19, 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Michael Kerrisk**  
January 19, 2013

Error reporting from the kernel (and low-level system libraries such as the C library) has been a primitive affair since the earliest UNIX systems. One of the consequences of this is that end users and system administrators often encounter error messages that provide quite limited information about the cause of the error, making it difficult to diagnose the underlying problem. Some recent discussions on the libc-alpha and Linux kernel mailing lists were started by developers who would like to improve this state of affairs by having the kernel provide more detailed error information to user space. 

The traditional UNIX (and Linux) method of error reporting is via the (per-thread) global `errno` variable. The C library wrapper functions that invoke system calls indicate an error by returning -1 as the function result and setting `errno` to a positive integer value that identifies the cause of the error. 

The fact that `errno` is a global variable is a source of complications for user-space programs. Because each system call may overwrite the global value, it is sometimes necessary to save a copy of the value if it needs to be preserved while making another system call. The fact that `errno` is global also means that signal handlers that make system calls must save a copy of `errno` on entry to the handler and restore it on exit, to prevent the possibility of overwriting a `errno` value that had previously been set in the main program. 

Another problem with `errno` is that the information it reports is rather minimal: one of somewhat more than one hundred integer codes. Given that the kernel provides hundreds of system calls, many of which have multiple error cases, the mapping of errors to `errno` values inevitably means a loss of information. 

That loss of information can be particularly acute when it comes to certain commonly used `errno` values. In [a message](<https://lwn.net/Articles/533097/>) to the libc-alpha mailing list, Dan Walsh explained the problem for two errors that are frequently encountered by end users: 

Traditionally, if a process attempts a forbidden operation, errno for that thread is set to EACCES or EPERM, and a call to strerror() returns a localized version of "Permission Denied" or "Operation not permitted". This string appears throughout textual uis and syslogs. For example, it will show up in command-line tools, in exceptions within scripting languages, etc. 

Those two errors have been defined on UNIX systems since early times. POSIX [defines](<http://pubs.opengroup.org/onlinepubs/9699919799/functions/V2_chap02.html#tag_15_03>) `EACCES` as ""an attempt was made to access a file in a way forbidden by its file access permissions"" and `EPERM` as ""an attempt was made to perform an operation limited to processes with appropriate privileges or to the owner of a file or other resource"". These definitions were fairly comprehensible on early UNIX systems, where the kernel was much less complex, the only method of controlling file access was via classical `rwx` file permissions, and the only kind of privilege separation was via user and group IDs and superuser versus non-superuser. However, life is rather more complex on modern UNIX systems. 

In all, `EPERM` and `EACCES` are returned by more than 3000 locations across the Linux 3.7 kernel source code. However, it is not so much the number of return paths yielding these errors that is the problem. Rather, the problem for end users is determining the underlying cause of the errors. The possible causes are many, including denial of file access because of insufficient (classical) file permissions or because of permissions in an ACL, lack of the right capability, denial of an operation by a Linux Security Module or by the [seccomp mechanism](<https://lwn.net/Articles/475043/>), and any of a number of other reasons. Dan summarized the problem faced by the end user: 

As we continue to add mechanisms for the Kernel to deny permissions, the Administrator/User is faced with just a message that says "Permission Denied" Then if the administrator is lucky enough or skilled enough to know where to look, he might be able to understand why the process was denied access. 

Dan's mail linked to a [wiki page](<https://fedoraproject.org/wiki/Features/FriendlyEPERM>) ("Friendly EPERM") with a proposal on how to deal with the problem. That proposal involves changes to both the kernel and the GNU C library (glibc). The kernel changes would add a mechanism for exposing a "failure cookie" to user space that would provide more detailed information about the error delivered in `errno`. On the glibc side, `strerror()` and related calls (e.g., `perror()`) would access the failure cookie in order obtain information that could be used to provide a more detailed error message to the user. 

Roland McGrath was quick [to point out](<https://lwn.net/Articles/533098/>) that the solution is not so simple. The problem is that it is quite common for applications to call `strerror()` only some time after a failed system call, or to do things such as saving `errno` in a temporary location and then restoring it later. In the meantime, the application is likely to have performed further system calls that may have changed the value of the failure cookie. 

Roland went on to identify some of the problems inherent in trying to extend existing standardized interfaces in order to provide useful error information to end users: 

It is indeed an unfortunate limitation of POSIX-like interfaces that error reporting is limited to a single integer. But it's very deeply ingrained in the fundamental structure of all Unix-like interfaces. 

Frankly, I don't see any practical way to achieve what you're after. In most cases, you can't even add new different errno codes for different kinds of permission errors, because POSIX specifies the standard code for certain errors and you'd break both standards compliance and all applications that test for standard errno codes to treat known classes of errors in particular ways. 

In response, Eric Paris, one of the other proponents of the failure-cookie idea [acknowledged](<https://lwn.net/Articles/533100/>) Roland's points, noting that since the standard APIs can't be extended, then changes would be required to each application that wanted to take advantage of any additional error information provided by the kernel. 

Eric subsequently posted [a note](<https://lwn.net/Articles/533101/>) to the kernel mailing list with a proposal on the kernel changes required to support improved error reporting. In essence, he proposes exposing some form of binary structure to user space that describes the cause of the last `EPERM` or `EACCES` error returned to the process by the kernel. That structure might, for example, be exposed via a thread-specific file in the `/proc` filesystem. 

The structure would take the form of an initial field that indicates the subsystem that triggered the error--for example, capabilities, SELinux, or file permissions--followed by a union of substructures that provide subsystem-specific detail on the circumstances that triggered the error. Thus, for a file permissions error, the substructure might return the effective user and group ID of the process, the file user ID and group ID, and the file permission bits. At the user-space level, the binary structure could be read and translated to human-readable strings, perhaps via a glibc function that Eric suggested might be named something like `get_extended_error_info()`. 

Each of the kernel call sites that returned an `EPERM` or `EACCES` error would then need to be patched to update this information. But, patching all of those call sites would not be necessary to make the feature useful. As Eric noted: 

But just getting extended denial information in a couple of the hot spots would be a huge win. Put it in capable(), LSM hooks, the open() syscall and path walk code. 

There were various comments on Eric's proposal. In response to concerns from Stephen Smalley that this feature might leak information (such as file attributes) that could be considered sensitive in systems with a strict security policy (enforced by an LSM), Eric [responded](<https://lwn.net/Articles/533105/>) that the system could provide a sysctl to disable the feature: 

I know many people are worried about information leaks, so I'll right up front say lets add the sysctl to disable the interface for those who are concerned about the metadata information leak. But for most of us I want that data right when it happens, where it happens, so It can be exposed, used, and acted upon by the admin trying to troubleshoot why the shit just hit the fan. 

Reasoning that its best to use an existing format and its tools rather than inventing a new format for error reporting, Casey Schaufler [suggested](<https://lwn.net/Articles/533114/>) that [audit](<http://people.redhat.com/sgrubb/audit/>) records should be used instead: 

the string returned by get_extended_error_info() ought to be the audit record the system call would generate, regardless of whether the audit system would emit it or not. If the audit record doesn't have the information you need we should fix the audit system to provide it. Any bit of the information in the audit record might be relevant, and your admin or developer might need to see it. 

Eric expressed [concerns](<https://lwn.net/Articles/533119/>) that copying an audit record to the process's `task_struct` would carry more of a performance hit than copying a few integers to that structure, concluding: 

I don't see a problem storing the last audit record if it exists, but I don't like making audit part of the normal workflow. I'd do it if others like that though. 

Jakub Jelinek [wondered](<https://lwn.net/Articles/533126/>) which system call Eric's mechanism should return information about, and whether its state would be reset if a subsequent system call succeeded. In many cases, there is no one-to-one mapping between C library calls and system calls, so that some library functions may make one system call, save `errno`, then make some other system call (that may or may not also fail), and then restore the first system call's `errno` before returning to the caller. Other C library functions themselves set `errno`. ""So, when would it be safe to call this new get_extended_error_info function and how to determine to which syscall it was relevant?"" 

Eric's [opinion](<https://lwn.net/Articles/533128/>) was that the mechanism should return information about the last kernel system call. ""It would be really neat for libc to have a way to save and restore the extended errno information, maybe even supply its own if it made the choice in userspace, but that sounds really hard for the first pass."" 

However, there are problems with such a bare-bones approach. If the value returned by `get_extended_error_info()` corresponds to the last system call, rather than the `errno` value actually returned to user space, this risks confusing user-space applications (and users). Carlos O'Donell, who had earlier [raised](<https://lwn.net/Articles/533130/>) some of the same questions as Jakub and pointed out the need to properly handle the extended error information when a signal handler interrupts the main program, [agreed](<https://lwn.net/Articles/533136/>) with Casey's [assessment](<https://lwn.net/Articles/533135/>) that `get_extended_error_info()` should always return a value that corresponds to the current content of `errno`. That implies the need for a user-space function that can save and restore the extended error information. 

Finally, David Gilbert [suggested](<https://lwn.net/Articles/533141/>) that it would be useful to broaden Eric's proposal to handle errors beyond `EPERM` and `EACESS`. ""I've wasted way too much time trying to figure out why mmap (for example) has given me an EINVAL; there are just too many holes you can fall into."" 

In the last few days, discussion in the thread has gone quiet. However, it's clear that Dan and Eric have identified a very real and practical problem (and one that has been [identified](<https://lwn.net/Articles/374794/>) by others in the past). The solution would probably need to address the concerns raised in the discussion--most notably the need to have `get_extended_error_info()` always correspond to the current value of `errno`--and might possibly also be generalized beyond `EPERM` and `EACCES`. However, that should all be feasible, assuming someone takes on the (not insignificant) work of fleshing out the design and implementing it. If they do, the lives of system administrators and end users should become considerably easier when it comes to diagnosing the causes of software error reports.
