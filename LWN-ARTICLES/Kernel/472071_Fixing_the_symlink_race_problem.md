---
title: Fixing the symlink race problem
url: https://lwn.net/Articles/472071/
date: "December 14, 2011"
category: "Security-Security technologies"
author: "By Jake Edge December 14, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
December 14, 2011

The problems with symbolic link race conditions have existed for decades, been well understood in that time, and developers have been given clear guidelines on how to avoid them. Yet they still persist, with new vulnerabilities discovered regularly. There is also a known way to avoid most of the problems by changing the kernel--something that has been done for many years in grsecurity and Openwall--but it has never made its way into the mainline. While kernel hackers are understandably unenthusiastic about working around buggy user-space programs in the kernel, this particular problem is severe enough that it probably makes sense to do so. It would seem that we are seeing some movement on that front. 

The basic problem is a time-to-check-to-time-of-use (TOCTTOU) flaw. Buggy applications will look for the existence and/or characteristics of temporary files before opening them. An attacker can exploit the flaw by changing the file (often by making a symlink) in between the check and the `open()`. If the program with the flaw has elevated privileges (e.g. setuid), and the attacker replaces the file with a symlink to a system file, serious problems can result. 

The bug generally happens in shared, world-writable directories that have the "sticky" bit set (like `/tmp`). The sticky bit on a directory is set to prevent users from deleting other users' files. So, the fix restricts the ability to follow symlinks in sticky directories. In particular, a symlink is only followed if it is owned by the follower or if the directory and symlink have the same owner. That solves much of the symlink race problem _without_ breaking any known applications. 

We [looked](<https://lwn.net/Articles/390323/>) at patches to restrict the following of symlinks in sticky directories in June 2010. Since that time, there has been a two-pronged approach, championed by Kees Cook, to try to get the code into the mainline. The first is the [Yama LSM](<https://lwn.net/Articles/464651/>), which is meant to collect up extensions to the Linux discretionary access control (DAC) model. But it runs afoul of the usual problem for specialized LSMs: the [inability to stack LSMs](<https://lwn.net/Articles/393008/>). 

Cook and others would clearly prefer to see the symlink changes go into the core VFS code, rather than via an LSM, but there has been a push by some to keep it out of the core. There was discussion of Yama and its symlink protections at the Linux Security Summit [LSM roundtable](<https://lwn.net/Articles/458652/>), where the plan to push Yama as a DAC enhancement LSM was hatched. That may well be a way forward, but Cook has also [posted](<https://lwn.net/Articles/470891/>) a patch set that would put the symlink restrictions into `fs/namei.c`. 

The latter patch attracted some interesting comments that would seem to indicate that Ingo Molnar and Linus Torvalds, at least, see value in closing the hole. None of the VFS developers have weighed in on this iteration, but Cook notes that the patch reflects feedback from Al Viro, which could be seen as a sign that he's not _completely_ opposed. [Molnar](<https://lwn.net/Articles/472141/>) was particularly unhappy that the hole still exists: 

Ugh - and people continue to get exploited from a preventable, fixable and already fixed VFS design flaw. 

Molnar also had some questions about the implementation, including whether the `PROTECTED_STICKY_SYMLINKS` kernel configuration parameter should default to 'yes', but was overall very interested in seeing the patch move forward. Torvalds had a [somewhat different take](<https://lwn.net/Articles/472144/>), ""Ugh. I really dislike the implementation."", but suggested a different mechanism to try to solve the underlying problem by using the permission bits on the symlink. His argument is that Cook's approach is not very ""polite"" because it is hidden away, so it turns into: 

some kind of hacky run-time random behavior depending on some invisible config option that people aren't even aware of. 

As Cook [points out](<https://lwn.net/Articles/472146/>), though, Torvalds's approach has its own set of ""weird hidden behaviors"". Torvalds admittedly had not thought his proposal through completely, but it does show an interest in seeing the problem solved. From Cook's perspective, the changes he is proposing simply change the behavior of sticky directories with respect to symlinks, whereas Torvalds's would have wider-ranging effects on symlink creation. Either might do the job, but Cook's solution does have an advantage in that the proposed changes have been shaken out for years in grsecurity and Openwall, as well as in Ubuntu more recently. 

Given that several high-profile kernel hackers seem to be in favor of fixing the problem--Ted Ts'o was also favorably disposed to a fix back in 2010--the winds may have shifted in favor of the core VFS approach. If Viro and the other VFS developers aren't completely unhappy with it, we could see it in 3.4 or so. 

If that were to happen, there is another related patch that would presumably also be pushed for mainline inclusion: hard link restrictions. That, like the symlink change, currently lives in Yama, though the case can be made that it should also be done in the core VFS code. That patch would disallow the creation of hard links to files that are inaccessible (neither readable nor writable) to the user making the link. It also disallows hard links to setuid and setgid files. That would close some further holes in the symlink race vulnerability, as well as fix some other application vulnerabilities. 

Should both the symlink and hard link restrictions make their way into the VFS core, that would only leave the `ptrace()` restrictions in Yama. Those restrictions allow administrators to disallow a process from using `ptrace()` on anything other than its descendants (unless it has the `CAP_SYS_PTRACE` capability). Currently, any process can trace any other running under the same UID, so a compromise in one running program could lead to disclosing credentials and other sensitive information from another running program. There may also be other DAC enhancements that Cook or others are interested in adding to Yama in the future. 

One way or another, the problem is severe enough that there should, at least, be a way for distributors or administrators to thwart these common vulnerabilities. Whether the fix lives in VFS or an LSM, providing an option to turn off a whole class of application flaws--which can often lead to system compromise--seems worth doing. Hopefully we are seeing movement in that direction.
