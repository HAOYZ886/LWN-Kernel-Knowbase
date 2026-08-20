---
title: "Symbolic links in \"sticky\" directories"
url: https://lwn.net/Articles/390323/
date: "June 2, 2010"
category: Security
author: "By Jake Edge June 2, 2010"
---

By **Jake Edge**  
June 2, 2010

Security problems that exploit badly written programs by placing symbolic links in `/tmp` are legion. This kind of flaw has existed in applications going back to the dawn of UNIX time, and new ones get introduced regularly. So a recent effort to change the kernel to avoid these kinds of problems would seem, at first glance anyway, to be welcome. But some kernel hackers are not convinced that the core kernel should be fixing badly written applications. 

These `/tmp` symlink races are in a class of security vulnerabilities known as time-of-check-to-time-of-use (TOCTTOU) bugs. For `/tmp` files, typically a buggy application will check to see if a particular filename exists and/or if the file has a particular set of characteristics; if the file passes that test, the program uses it. An attacker exploits this by racing to put a symbolic link or different file in `/tmp` between the time of the check and the open or create. That allows the attacker to bypass whatever the checks are supposed to enforce. 

For programs with normal privilege levels, these attacks can cause a variety of problems, but don't lead to system compromise. But for setuid programs, an attacker can use the elevated privileges to overwrite arbitrary files in ways that can lead to all manner of ugliness, including complete compromise via privilege escalation. There are various [guides](<http://www.dwheeler.com/secure-programs/Secure-Programs-HOWTO.html#TEMPORARY-FILES>) that describe how to avoid writing code with this kind of vulnerability, but the flaw still gets reported frequently. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Ubuntu security team member Kees Cook [proposed](<https://lwn.net/Articles/390565/>) changing the kernel to avoid the problem, not by removing the race, but by stopping programs from following the symlinks that get created. "Proper" fixes in applications will completely avoid the race by creating random filenames that get opened with `O_CREAT|O_EXCL`. But, since these problems keep cropping up after multiple decades of warnings, perhaps another approach is in order. Cook adapted code from the Openwall and grsecurity kernels that did just that. 

Since the problems occur in shared directories (like `/tmp` and `/var/tmp`) which are world-writable, but with the "sticky bit" turned on so that users can only delete their own files, the patch restricts the kinds of symlinks that can be followed in sticky directories. In order for a symlink in a sticky directory to be followed, it must either be owned by the follower, or the directory and symlink must have the same owner. Since shared temporary directories are typically owned by root, and random attackers cannot create symlinks owned by root, this would eliminate the problems caused by `/tmp` file symlink races. 

The first version of the patch elicited a few suggestions, and an ACK by Serge Hallyn, but no complaints. Cook obviously did a fair amount of research into the problem and anticipated some objections from earlier linux-kernel discussions, which he linked to in the post. He also linked to a [list](<http://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=/tmp>) of 243 CVE entries that mention `/tmp`--not all are symlink races, but many of them are. When Cook revised and [reposted](<http://lwn.net/Articles/390009/>) the patch, though, a few complaints cropped up. 

For one thing, Cook had anticipated that VFS developers would object to putting his test into that code, so he put it into the capabilities checks (`cap_inode_follow_link()`) instead. That didn't sit well with Eric Biederman, who [said](<https://lwn.net/Articles/390612/>): 

Placing this in cap_inode_follow_link is horrible naming. There is nothing capabilities about this. Either this needs to go into one or several of the security modules or this needs to go into the core vfs. 

Alan Cox [agreed](<https://lwn.net/Articles/390615/>) that it should go into SELinux or some specialized Linux security module (LSM). He also suggested that giving each user their own `/tmp` mountpoint would solve the problem as well, without requiring any kernel changes: ""Give your users their own /tmp. No kernel mods, no misbehaviours, no weirdomatic path walking hackery. No kernel patch needed that I can see."" 

But Cook and others are not convinced that there are any legitimate applications that require the ability to follow these kinds of symlinks. Given that following them has been a source of serious security holes, why not just fix it once and for all in the kernel? One could argue that changing the behavior would violate the POSIX standard--one of the objections Cook anticipated--but that argument may be a bit weak. Ted Ts'o [believes](<https://lwn.net/Articles/390616/>) that POSIX doesn't really apply because the sticky bit isn't in the standard: 

So for people who to argue against this (which I believe to be a useful restriction, and not one that necessarily has to be done in a LSM), it's not sufficient to say that it is a POSIX violation, because it isn't. The sticky bit itself wasn't originally considered by POSIX, and many systems which implemented the sticky bit had no problems becoming [certified] as POSIX compliant. 

Per-user `/tmp` directories might solve the problem, but come with an administrative burden of their own. Eric Paris [notes](<https://lwn.net/Articles/390620/>) that it might be a better solution, but it doesn't come for free: 

Now, if we used filesystem namespaces regularly for years and users, administrators, and developers dealt with them often I agree that would probably be the preferred solution. It would solve this issue, but in introduces a whole host of other problems that are even more obvious and even likely to bite people. 

Ts'o [agrees](<https://lwn.net/Articles/390621/>): ""I do have a slight preference against per-user /tmp mostly because it gets confusing for administrators, and because it could be used by rootkits to hide themselves in ways that would be hard for most system administrators to find."" Based on that and other comments, Cook [revised](<https://lwn.net/Articles/390624/>) the patches again, moving the test into VFS, rather than trying to come in through the security subsystem. 

In addition, he changed the code so that the new behavior defaulted "off" to address one of the bigger objections. Version 3 of the patch was posted on June 1, and has so far only seen [comments](<https://lwn.net/Articles/390629/>) from Al Viro, who doesn't seem convinced of the need for the change, but was nevertheless discussing implementation details. 

It may be that Viro and other filesystem developers--Christoph Hellwig did not seem particularly in favor of the change for example--will oppose this change. It is, at some level, a band-aid to protect poorly written applications, but it also provides a measure of protection that some would like to have. As Cook [pointed out](<https://lwn.net/Articles/390631/>), the Ubuntu kernel already has this protection, but he would like to see that protection extended to all kernel users. Whether that happens remains to be seen.
