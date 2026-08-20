---
title: "Kernel Summit 2006: Security"
url: https://lwn.net/Articles/191737/
date: "July 19, 2006"
category: "AppArmor; Modules-Security modules; SELinux; Security-Security technologies"
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
The security session, led by Chris Wright, SELinux developer Stephen Smalley and AppArmor developer Tony Jones, was given the task of addressing two questions: (1) what is required to merge AppArmor, if that is ever to happen, and (2) should the Linux security module (LSM) interface be removed? There was more progress on the second question than the first. 

The LSM interface has been in the mainline for five years now, and there is still only one in-tree user: SELinux. That fact might argue for the removal of LSM as an unneeded abstraction layer, but there are other security policies out there which have been proposed for inclusion. At the head of the line is AppArmor, which was recently acquired and open-sourced by Novell. Tony Jones argued that this code is shipped with SUSE Linux, and it is used by real people who actually like it. It is, he thinks, a suitable addition to the kernel. 

AppArmor, says Tony, is focused on network-based threats. It is intended to protect servers, and to enable fast, easy policy development. This claim drew the first objection: Christoph Hellwig noted that security is never easy. Rather than trying to come up with an easy fix via a security framework, he said, developers should be getting server applications right and "fix the crappy code." The response was that crappy code will always exist, but that it would be nice to have reasonably secure systems anyway. 

The real issue with AppArmor, however, is its use of pathnames in the definition of security policies. To many people in the security field, pathnames are an insecure mechanism for this task. To drive this point home, Stephen Smalley put up a slide entitled "pathname-based mechanism considered harmful"; his purpose was clearly to oppose the inclusion of AppArmor. Anything based on pathnames, says Stephen, is inherently nondeterministic - not a nice feature in a security system. 

Not everybody agrees with that point of view, however. In the end, pathnames are how people and programs find files - in some way, they _are_ the file just as much as the inode is. A program looking for account information will open `/etc/passwd` without caring much which inode is behind it. Developers who see things this way (and Linus is one of them) do not think that AppArmor should be blocked because of its use of pathnames. 

Besides, there is a way to address the biggest objection to pathname-based policies: that they can be circumvented by creating a new link (and, thus, a different path) to a restricted binary. If the ability to create such links is controlled, the problem goes away. AppArmor does attempt to impose this sort of control; it could be strengthened by adding a "links to this file cannot be created" permission bit somewhere. 

As one might expect, this discussion went around the room for some time without generating much consensus. There is a sizeable group of kernel developers who feel that AppArmor should be allowed into the kernel. Linux supports an amazing variety of filesystems, some of which were described as "demented." Certainly there must be room for a security policy which some people find useful. Others, however, remain strongly in opposition to the merging of AppArmor. 

Toward the end, attention was forcibly redirected to the problem of what to do about the LSM framework. Stephen noted that SELinux has little use for LSM, that things could work just as well without it. Tony said that AppArmor needed LSM, that there was no way to implement it on top of the lower layers of SELinux, as has been suggested by some developers. Linus said that the entire reason that LSM had been added was to avoid having this sort of discussion every year. The fact that the security policy discussion only returned when the removal of LSM was threatened sort of backs up that point. 

The end result, as expressed by Linus, is that LSM will remain in the kernel. Since there is no prospect of a consensus on the One True Security Policy, there needs to be a way to support more than one of them. A more interesting discussion might have been the often-threatened rewrite of LSM to provide a better, more restricted set of operations, but there was no time for that here. 

  * [Next: Paravirtualization and containers](<https://lwn.net/Articles/191923/>).
