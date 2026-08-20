---
title: Writing your own security module
url: https://lwn.net/Articles/674949/
date: "February 10, 2016"
category: "Modules-Security modules; Security-Security modules"
author: "By Jonathan Corbet February 10, 2016 linux.conf.au 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 10, 2016

* * *

[linux.conf.au 2016](<https://lwn.net/Archives/ConferenceByYear/#2016-linux.conf.au>)

Casey Schaufler started off his [linux.conf.au](<http://linux.conf.au/>) 2016 talk by noting that a prospective security-module author may be told that writing a new module is not necessarily a good idea. They'll hear that we already have a good selection of security modules, or it can all be done with SELinux anyway and, besides, kernel programming is hard. But, he said, there are indeed good reasons for wanting to write a new security module; he was there to discuss those reasons and give some guidance for those unfamiliar with this part of the kernel. 

So why might one want to write a new security module? Our current modules, he said, are showing their age; they follow [a design](<https://en.wikipedia.org/wiki/Bell%E2%80%93LaPadula_model>) that dates from a time when users actually sat in a computer room to do their work. These modules are a poor fit to the concerns we have now; they were not designed with systems like smartphones in mind. We need to start working on ideas that don't date from the era of paper tape. And, in any case, there _are_ things that cannot be done with SELinux. 

It is important to understand what security modules can do. They are a way to add new restrictive controls to operations that a process might try to perform; they cannot replace the existing discretionary access control checks, which will be done anyway. Thus, they offer a new opportunity to say "no" to an operation, but they cannot authorize an action that the user would not have otherwise been able to perform. 

There are a few basic rules to bear in mind when contemplating a new security module. The first of those is to avoid duplicating existing modules. If SELinux can do what you need, you're better off joining its community and pushing things in the direction that you need. Another rule is to not rely heavily on user-space helpers. There is a proprietary module out there, he said, that asks user space to make the actual security decisions. This solution is inefficient and provides a hook for proprietary code; either reason would be enough to keep it from ever going upstream. Remember that you're playing in kernel space, so try not to upset the kernel developers; he named Al Viro in particular as one not to inflame. 

The most important rule, though, is to plagiarize freely from other security modules; don't reinvent things that already exist and work. When he wrote [Smack](<https://lwn.net/Articles/244531/>), he took a lot of things from SELinux. That, he said, is how we do things in the Linux community. 

#### How security modules work

Casey started into the mechanics of security modules by talking about the hooks that they rely on. There are function calls scattered around the kernel that exist to give security modules a chance to make a decision on a specific action; their names all start with "`security_`". Modules use these hooks to do access checks and perform general data management; it is only necessary to write the hooks that are relevant to the task at hand. Following normal kernel conventions, hooks return zero for success (generally allowing the requested operation); `EACCES` should be used to indicate a policy denial, while `EPERM` means that a necessary privilege is missing. 

Many (or most) hooks are object-based, in that they relate to a specific object in kernel space. Hooks often deal with `inode` structures, for example, since those structures represent files within the kernel. Some hooks, though, work with pathnames instead. Paths are more human-friendly, since that's how people deal with files, but they may not uniquely identify the underlying object (files can have more than one name, for example). 

"Security blobs" are data structures attached to objects by a security module. One will often see the terms `secctx`, which refers to a text string associated with a blob, and `secid`, a 32-bit ID number for a blob. There are two types of modules, called "major" and "minor," with the difference being that major modules use blobs. There can only be [![\[Casey Schaufler\]](https://static.lwn.net/images/conf/2016/lca/CaseySchaufler-sm.jpg)](<https://lwn.net/Articles/674955/>) one major module active in the kernel, and it runs after any other module. There is no mechanism, yet, for sharing the security-blob pointers between modules; Casey allowed as to how fixing that is an [ongoing crusade](<https://lwn.net/Articles/518345/>) of his. Minor modules, thus, have no blobs and don't maintain any per-object state. They run after the discretionary checks, but before the major module, if one exists. 

There are some questions that should be answered before attempting to design a new security module, starting with: what resource is to be protected by the module? If you can't answer that, Casey said, you're not thinking about security. Answers might be files created by a particular user, or specific paths within the filesystem, or one might want to protect specific processes from each other. 

That leads to the second question: what is that resource to be protected from? Traditionally, security has been based around users, but now we think about things like malicious apps. So, rather than protecting users from each other, we're now concerned with protecting Facebook's data from Netflix. 

Finally, a security-module developer needs to know how that protection will be done. The traditional answer is to simply deny access to unauthorized users, but other approaches are possible. Logging of such attempts is often done, of course. One could consider more wide-ranging changes, such as changing the ownership of a file to match the last user who wrote to it via group access. 

#### Various details

Security modules can make information available under `/proc`. One should, however, resist the temptation to reuse the attribute names already used by SELinux; a new security module should make its own names. 

Objects contain numerous attributes that can be used by a security module; these include ownership, access mode, object types and sizes, etc. Modules can use them any way they like, but they should not change their fundamental meaning. The user ID should not identify the application, as "a certain system" has done. Security modules can make decisions based on pathnames if that works best, though the interface to pathnames inside the kernel is not the most convenient. 

With regard to networking, Casey said, there may not be much for a security-module developer to do. Linux has the netfilter subsystem that can make all kinds of access-control decisions; that approach should always be tried first. If a module must be written, there are hooks for various socket operations, and for packet delivery as well. The `SO_PEERSEC` socket operation can be used to pass security attributes to another process. Working with Unix-domain sockets is easy, he said, because the security module has access to both ends of the connection. Internet sockets are harder, since only one end is available. The [CIPSO](<https://lwn.net/Articles/204905/>) mechanism can be used to send attributes across a link; support for [CALIPSO](<https://tools.ietf.org/html/rfc5570>), which will make similar functionality available under IPv6, is coming. 

Casey suggested that modules probably want to log access denials, to help with policy debugging if nothing else. Helpful stuff can be found in `<linux/lsm_audit.h>`. The actual data to be logged is up to the module author; various utilities are available for formatting that data in user space. 

Non-trivial modules probably need control interfaces; administrators will want to be able to change access rules, look at statistics, and more. Casey advised against adding new `ioctl()` calls or system calls; instead, a virtual filesystem should be created. A call to `sysfs_create_mount_point()` makes that easy; he recommended borrowing the relevant code from an existing module. 

Finally, what about stacking of security modules? At this point, stacking of minor modules is easy and supported, but there can only be one major module at a time, since there is only one security-blob pointer. There is, he said, a way to cheat on systems where the kernel is built from source: simply add the new security blob to the SELinux blob. Work is in progress to allow multiple major modules, but they cannot be supported now. 

In the end, Casey said, anybody looking to write a security module should have a good reason for doing it. Some stuff, after all, really does belong in user space. If you write a module, do it properly: provide documentation and support the code. Don't reinvent the wheel; generic security has long since been done, show us something new instead. Nobody has done a good application resource-management security policy, for example. There is interesting potential around policies tied to the sensors found on current devices. Security, he said, does not have to be dull. 

The [video of this talk](<http://mirror.linux.org.au/linux.conf.au/2016/03_Wednesday/D4.303_Costa_Theatre/How_To_Write_A_Linux_Security_Module_That_Makes_Sense_For_You.webm>) is available on the LCA site. 

[Your editor thanks LCA for assisting with his travel expenses.]
