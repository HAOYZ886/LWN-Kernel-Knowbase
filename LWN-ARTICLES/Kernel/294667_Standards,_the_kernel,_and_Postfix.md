---
title: "Standards, the kernel, and Postfix"
url: https://lwn.net/Articles/294667/
date: "August 20, 2008"
category: Symbolic links
author: "By Jake Edge August 20, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
August 20, 2008

Standards like POSIX are meant to make life easier for application developers by providing rules on the semantics of system calls for multiple different platforms. Sometimes, though, operating system developers decide to change the behavior of their platform--with full knowledge that it breaks compatibility--for various reasons. This requires application developers to notice the change and take appropriate action; not doing so can lead to a security hole like the one found in the Postfix mail transfer agent (MTA) recently. 

The behavior of links, created using the `link()` system call--on Linux, Solaris, and IRIX--is what tripped up Postfix. In particular, what happens when a hard link is made to a symbolic link. Many long-time UNIX hackers don't realize that you can even do that, nor what to expect if you do. Postfix relied on a particular, standard-specified behavior that many operating systems, including early versions of Linux, follow. 

Links can be a somewhat confusing, or possibly unknown, part of UNIX-like filesystems, so a bit of explanation is in order. A link created with `link()`--also known as a hard link--is an alias for a particular file. It simply gives an additional name by which a particular chunk of bytes on the disk can be referenced. For example: 

```
link("/path/to/foo", "/link/to/foo");
```

creates a second entry in the filesystem (called `/link/to/foo`) which points to the same file as `/path/to/foo`. The file being linked to must exist and reside on the same filesystem as the link. 

Symbolic links, on the other hand, are aliases of a different sort. A symbolic link creates a new entry (e.g. inode) in the filesystem which contains the path of the linked-to file as a string. There is no requirement that the file exist or be on the same filesystem--the only real requirement is that the path conform to standard pathname rules. The `symlink()` system call is used to create them: 

```
symlink("/path/to/foo", "/symlink/to/foo");
```

Both symbolic links and hard links can also be created from the command line using the `ln` command (adding a `-s` option for symbolic links). 

So, when making a hard link to a symbolic link, there are two choices: either follow the symbolic link to its, possibly nonexistent, target and link to that or link to the symbolic link inode itself. POSIX requires that the symbolic link be fully resolved to an actual existing file, which is the behavior that Postfix relies upon. 

The exact sequence of events is lost in the mists of time, but Linux changed to non-standard behavior--at least partially for compatibility with Solaris--in kernel version 1.3.56 (which was released in January 1996). Some [discussion](<
http://groups.google.com/group/comp.os.linux.development.system/msg/7b3eadf132e67190?>) prior to that change adds an additional reason for it: user space has no way to make a link to a symbolic link without it. Some saw that as a flaw in the interface and proposed the change. An application developer that wanted the original behavior would be able to implement it by resolving any symbolic links before making the hard link. 

To further complicate things, it appears that the POSIX behavior was restored in the 2.1 development series, only to be [changed back](<http://lkml.org/lkml/1998/9/10/4>) in late 1998. This change led to the comments currently in `fs/namei.c` for the function implementing `link()`: 

```
/*
         * Hardlinks are often used in delicate situations.  We avoid
         * security-related surprises by not following symlinks on the
         * newname.  --KAB
         *
         * We don't follow them on the oldname either to be compatible
         * with linux 2.0, and to avoid hard-linking to directories
         * and other special files.  --ADM
         */
```

Where `oldname` is the file being linked to and `newname` is the name being created. For the curious, KAB is Kevin Buhr and ADM is Alan Modra. 

Unfortunately, according to Postfix author Wietse Venema, the `link(2)` man page didn't change until sometime in 2006. This makes it fairly difficult for application developers to learn about the change, especially because they may not follow kernel development closely. 

Postfix allows root-owned symbolic links to be used as the target for local mail delivery, specifically to handle things like `/dev/null` on Solaris, which is a symbolic link. Because an attacker can make a link to a root-owned symbolic link on vulnerable systems, Postfix can get confused and deliver mail to files that it shouldn't. This can lead to privilege escalation (via executing code as root) by making a hard link to a symbolic link of an init script ([CVE-2008-2936](<http://nvd.nist.gov/nvd.cfm?cvename=CVE-2008-2936>)). 

As Venema outlines in the [Postfix security advisory](<http://archives.neohapsis.com/archives/postfix/2008-08/0392.html>), the problem can be resolved by requiring that symbolic links used for local delivery reside in a directory that is only writeable by root. It is not a perfect solution, though: ""This change will break legitimate configurations that deliver mail to a symbolic link in a directory with less restrictive permissions."" There are other workarounds for people who don't want to use the provided patch to Postfix. Protecting the mail spool directory is one solution; Venema provides a script to use to do that. Some systems can be configured to disallow links to files owned by others, which is another way to avoid the problem. 

This issue has given Postfix a bit of a black eye, but that is rather unfair. The problem was found by a SUSE code inspection, but it has existed in certain kinds of Linux installations of Postfix for a long time. It could be argued that testing should have found it--there is a simple test for vulnerable systems--but relying on documented behavior that is part of an important standard that Linux is said to support is not completely unreasonable either. It is likely that the full implications of not supporting the standard were not completely understood until recently. 

Linux was still in its infancy when the original change went in. One would like to think that a change of that type today would be nearly impossible because it breaks the kernel's user-space interface. If it were to happen, somehow, the resulting hue and cry would be loud enough that application developers would hear. But that's for intentional changes; a bug introduced into a dark corner of the kernel's API might go unnoticed for quite some time. Hopefully, none of those lingers for ten years before being discovered. 

**Update:** The original article referred to [CVE-2008-2937](<http://nvd.nist.gov/nvd.cfm?cvename=CVE-2008-2937>) as also being a consequence of the link issue, which it is not. It is an unrelated issue that was found during the same code review.
