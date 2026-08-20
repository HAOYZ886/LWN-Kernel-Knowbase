---
title: "TOMOYO Linux and pathname-based security"
url: https://lwn.net/Articles/277833/
date: "April 14, 2008"
category: "AppArmor; Modules-Security modules; Security-Security modules; TOMOYO Linux"
author: "By Jonathan Corbet April 14, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 14, 2008

It takes a certain kind of courage to head down a road when one can plainly see the unpleasant fate which befell those who went before. So one might think that the fate of AppArmor would deter others from following a similar path. The developers of [TOMOYO Linux](<http://elinux.org/TomoyoLinux>) are not easily put off, though. Despite having a security subsystem which shares a number of features with AppArmor, these developers are pushing forward in an attempt to get their code into the mainline. 

AppArmor, remember, is a Linux security module which uses pathnames to make security decisions. So it is entirely conceivable that two different security policies could apply to the same file if that file is accessed by way of two different names. This approach helps make AppArmor easier to administer than SELinux, but it has given AppArmor major problems in the review process for a few reasons: 

  * There has been strong resistance to the addition of any new security modules at all, to the point that proposals to remove the LSM framework altogether have been floated. 

  * Some security developers see a pathname-based mechanism as being fundamentally insecure. SELinux developers, in particular, have been very strongly against pathname-based security. To these developers, security policies should apply directly to objects (or to labels attached directly to objects) rather than to names given to objects. 

  * The current Linux security module hooks, not being developed with pathname-based security in mind, do not provide sufficient information to the low-level file operation hooks. So AppArmor had to reconstruct pathnames within its security hooks. The method chosen for this reconstruction was, one might say, not universally admired. 

If the TOMOYO Linux developers are serious about getting their code into the mainline, they will need to have answers to these objections. 

As it happens, the first two obstructions have mostly gone away. Casey Schaufler's persistence finally resulted in the merging of the SMACK security module for 2.6.25; it is the only such module, other than SELinux, ever to get into the mainline. Now that SMACK has paved the way, talk of removing the LSM framework (which had been strongly vetoed by Linus in any case) has ended and the next security module should have an easier time of it. 

Linus has also decreed that pathname-based security modules are entirely acceptable for inclusion into the kernel. So, while some developers remain highly skeptical of this approach, their skepticism cannot, on its own, be used as a reason to keep a pathname-based security module out. Pathname-based approaches appear to be "secure enough" for a number of applications, and there are [some advantages](<https://lwn.net/Articles/277842/>) to using that approach. 

All of the above is moot, though, if the TOMOYO Linux developers are unable to implement pathname-based access control in a way which passes muster. The [recent TOMOYO Linux patch](<http://lwn.net/Articles/276603/>) took a different approach to this problem: since the LSM hooks do not provide the needed information, the developers just added a new set of hooks, outside of LSM, for use by TOMOYO Linux. And, while they were at it, they added new hooks at all enforcement points. This was not a popular decision, to say the least. The whole idea behind LSM was to have a single set of hooks for all security modules; if every module now adds its own set of hooks, that purpose will have been defeated and the kernel will turn into a big mess of security hooks. Duplicating the LSM framework is not the way to get a security module into the mainline. 

So, somehow, the TOMOYO Linux developers will need to implement pathname-based security in a different way. The most obvious thing to do would be to modify the existing hooks to supply the requisite information (being a pointer to the `vfsmount` structure). The problem here is that, at the point where the LSM hooks are called, that structure is not available; it is only used at the higher levels of the virtual filesystem code. So either some core VFS functions would have to be changed (so the `vfsmount` pointer could be passed into them), or a new set of hooks would need to be placed at a level where that pointer is available. It [appears](<https://lwn.net/Articles/277846/>) that the second approach - adding new hooks in the namespace code - will be taken for the next version of the patch. 

As the TOMOYO Linux developers work through this problem, they are likely to be closely watched by the (somewhat reduced in number) AppArmor group. There appears to be a resurgence of interest in getting AppArmor merged, so we will probably see AppArmor put forward again in the near future. That will be even more likely if TOMOYO Linux is able to solve the pathname problem in a way which survives review and gets into the kernel.
