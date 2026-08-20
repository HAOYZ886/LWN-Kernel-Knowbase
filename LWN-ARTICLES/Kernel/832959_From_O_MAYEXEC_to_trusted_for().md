---
title: From O_MAYEXEC to trusted_for()
url: https://lwn.net/Articles/832959/
date: "October 1, 2020"
category: "Filesystems-Virtual filesystem layer; Security-Language interpreters"
author: "By Jonathan Corbet October 1, 2020"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 1, 2020

The ability to execute the contents of a file is controlled by the execute-permission bits — some of the time. If a given file contains code that can be executed by an interpreter — such as shell commands or code in a language like Perl or Python, for example — there are easy ways to run the interpreter on the file regardless of whether it has execute permission enabled or not. Mickaël Salaün has been working on tightening up the administrator's control over execution by interpreters for some time, but has struggled to find an acceptable home for this feature. His latest attempt takes the form of a new system call named `trusted_for()`. 

Tightly locked-down systems are generally set up to disallow the execution of any file that has not been approved by the system's overlords. That control is nearly absolute when it comes to binary machine code, especially when security modules are used to enforce signature requirements and prevent techniques like mapping a file into some process's address space with execute permission. Execution of code by an interpreter, though, just looks like reading a file to the kernel so, without cooperation from the interpreter itself, the kernel cannot know whether an attempt is being made to execute code contained within a given file. As a result, there is no way to apply any kernel-based policies to that type of access. 

Enabling that cooperation is the point of Salaün's work; it is, at its core, a way for an interpreter to inform the kernel that it intends to execute the contents of a file. Back in May 2020, the first attempt tried to add [an `O_MAYEXEC` flag](<https://lwn.net/Articles/820000/>) to be used with the [`openat2()`](<http://man7.org/linux/man-pages/man2/openat2.2.html>) system call. If system policy does not allow a given file to be executed, an attempt to open it with `O_MAYEXEC` will fail. 

This feature was controversial for a number of reasons, but Salaün persisted with the work; [version 7 of the `O_MAYEXEC` patch set](<https://lwn.net/ml/linux-kernel/20200723171227.446711-1-mic@digikod.net/>) was posted in August. At that point, Al Viro [asked](<https://lwn.net/ml/linux-kernel/20200727042106.GB794331@ZenIV.linux.org.uk/>), in that special way he has, why this check was being added to `openat2()` rather than being made into its own system call. Florian Weimer [added](<https://lwn.net/ml/linux-kernel/87y2n55xzv.fsf@oldenburg2.str.redhat.com/>) that doing so would allow performing checks on an already-open file; that would enable interpreters to indicate an intent to execute code read from their standard input, for example — something that `O_MAYEXEC` cannot do. Salaün [replied](<https://lwn.net/ml/linux-kernel/eaf5bc42-e086-740b-a90c-93e67c535eee@digikod.net/>) that controlling the standard input was beyond the scope of what he was trying to do. 

Nonetheless, he tried to address this feedback in [version 8](<https://lwn.net/ml/linux-kernel/20200908075956.1069018-1-mic%40digikod.net/>), which implemented a new flag (`AT_INTERPRETED`) for the proposed [`faccessat2()`](<https://lwn.net/ml/linux-kernel/20200416143532.11743-1-mszeredi@redhat.com/>) system call instead. That allowed the check to be performed on either a file or an open file descriptor. This attempt did not fly either, though, with Viro [insisting](<https://lwn.net/ml/linux-kernel/20200909171316.GW1236603@ZenIV.linux.org.uk/>) that a separate system call should be provided for this feature. This approach also introduced a potential race condition if an attacker could somehow change a file between the `faccessat2()` call and actually opening the file. So Salaün [agreed](<https://lwn.net/ml/linux-kernel/2ed377c4-3500-3ddc-7181-a5bc114ddf94@digikod.net/>) to create a new system call for this functionality. 

Thus, [the ninth version](<https://lwn.net/ml/linux-kernel/20200910164612.114215-1-mic@digikod.net/>) introduced `introspect_access()`, which would ask the kernel if a given action was permissible for a given open file descriptor. There comes a point in kernel development (and beyond) when one can tell that the substantive issues have been addressed: when everybody starts arguing about naming instead. That is what happened here; Matthew Wilcox [didn't like the name](<https://lwn.net/ml/linux-kernel/20200910170424.GU6583@casper.infradead.org/>), saying that checking policy on a file descriptor is not really "introspection". Various suggestions then flew by, including `security_check()`, `interpret_access()`, `entrusted_access()`, `fgetintegrity()`, `permission()`, `lsm()`, `should_faccessat()`, and more. 

In [the tenth version](<https://lwn.net/ml/linux-kernel/20200924153228.387737-1-mic@digikod.net/>), posted on September 24, Salaün chose none of those. The proposed new system call is now: 

```
int trusted_for(const int fd, const enum trusted_for_usage usage,
                        const unsigned int flags);
```

The `fd` argument is, of course, the open file descriptor, while `usage` describes what the caller intends to do with that file descriptor; in this patch set, the only possible option is `TRUSTED_FOR_EXECUTION`, but others could be added in the future. There are no flags defined, so the `flags` argument must be zero. The return value is zero if the system's security policy allows the indicated usage, or `EACCES` otherwise. In the latter case, it is expected that the caller will refuse to proceed with executing the contents of the file. 

The patch also adds a new sysctl knob called `fs.trust_policy` for setting a couple of policy options. Setting bit zero disables execution access for files located on a filesystem that was mounted with the `noexec` option; bit one disables execution for any file that does not have an appropriate permission bit set. Both of these are checks that are not made by current kernels. There are no extra security-module hooks added at this time, but that would appear to be the plan in the future; that will allow more complex policies and techniques like signature verification to be applied. 

This time around, even the name of the system call has avoided complaints — as of this writing, at least. So it may just be that this long-pending feature will finally make its way into the mainline kernel. That is not a complete solution to the problem, of course. Security-module support will eventually be needed, but also support within the interpreters themselves. That will require getting patches accepted into a variety of user-space projects. Fully locking down access to files by interpreters, in other words, is going to take a while yet.
