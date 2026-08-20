---
title: "FBAC-LSM"
url: https://lwn.net/Articles/369609/
date: "January 13, 2010"
category: "Security-Security modules"
author: "By Jake Edge January 13, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
January 13, 2010

Process sandboxes for security are getting lots of attention these days. There are standalone utilities like [isolate](<http://lwn.net/Articles/368690/>) and [Rainbow](<http://wiki.laptop.org/go/Rainbow>), sandboxes integrated into applications like the [Chromium sandbox](<http://lwn.net/Articles/347547/>), as well as tools that use existing LSMs such as the [SELinux sandbox](<http://danwalsh.livejournal.com/28545.html>). Furthermore, there are various proposals floating around to add Linux kernel features in support of application sandboxes, such as the [seccomp additions](<http://lwn.net/Articles/332974/>) and [network restrictions](<http://lwn.net/Articles/368730/>). An LSM specifically designed for application sandboxing, which uses a new model called Functionality-Based Application Confinement (FBAC), was [introduced](<http://lwn.net/Articles/366468/>) on linux-kernel back in December. 

[FBAC-LSM](<http://schreuders.org/FBAC-LSM>) came out of Z. Cliffe Schreuders's PhD research, and is a prototype implementation of the FBAC model. It uses an earlier version of the LSM interface, with the AppArmor pathname-based hooks and still needs ""quite a bit of work to be done before it is ready for production systems or formal code review."" Schreuders is looking for collaborators to work on completing the project, presumably with an eye towards getting it into the mainline. 

The basic idea behind FBAC is to make security policy more accessible and understandable to users, so that application restrictions are more widely adopted. A major component of the FBAC system is a GUI-based policy manager that can guide users through setting policies for particular applications. Users specify the high-level needs of an application based on its type (such as web browser or file editor) and the policy manager will help create the policies that will govern its behavior. 

In developing the policy manager, Schreuders analyzed over a hundred different applications to extract common behaviors that could be encapsulated in the FBAC policies. This allows the policy manager to automate certain aspects of developing policies for new applications, including things like configuration files, network ports, and other resources that the application requires. The policy manager also has a "learning mode" where it can observe the application and suggest additional privileges that might be granted. 

FBAC has the underlying concept of "functionalities", which are essentially a set of permissions for file and network operations that are allowed. These are fine-grained permissions for things like "file_read", "file_getattr", "file_execute", "dir_mkdir", "network_incoming", etc. The permissions which are granted to a particular functionality are listed in its definition. 

Functionalities are hierarchical, so that they can incorporate other, lower-level permissions into one that governs an entire application or class of applications. In addition, they are parameterized so that a functionality can be applied to multiple different applications, with the parameters specifying the particular files, directories, and network destinations that the permissions are granted for. 

Both mandatory access control (MAC) and discretionary access control (DAC) are supported by FBAC. Application policy can be permanently set by an administrator, so that an ordinary user cannot make changes, or FBAC can be configured to allow users to further restrict applications beyond the policies set by the administrator. The confinement of an application then depends on the intersection of these mandatory and discretionary policies. 

Allowing users to specify reduced privileges for arbitrary applications risks running afoul of the problems with `setuid()` programs that other sandboxing mechanisms (the network restrictions feature mentioned above for example) have encountered. Some means of stopping unprivileged users from changing the environment expected by `setuid()` programs will need to be provided. 

The interface to FBAC-LSM is via a filesystem which is mounted at `/sys/kernel/security/fbac-lsm`. Various files in the directory allow querying the existing installed policies as well as adding new ones. There are several steps to sending the policy information, with each piece being written to a separate file in the directory. That is followed by "commit" being written to `/sys/kernel/security/fbac-lsm/commit`, which actually causes the policy to be processed. That is rather race-prone, but is required by the `sysfs` "one value per file" rule. It seems likely that FBAC-LSM will eventually change its interface to a private filesystem like those used by Smack and SELinux. 

FBAC is a different approach from that taken by other security solutions, but it has enough similarities that Schreuders has plans to make the policy manager read and write AppArmor and [SEEdit](<http://seedit.sourceforge.net/>) policies. But FBAC definitely lives up to its prototype billing. The code is rather disorganized and littered with commented-out sections that make it somewhat hard to follow. 

The current incarnation of FBAC-LSM certainly has the feel of code that was put together somewhat hurriedly for a PhD dissertation, rather than as a "real" LSM. But it does embody some interesting ideas that merit further attention. One of the biggest hurdles faced by various security solutions (for which SELinux is the poster child) is the complexity of developing and--more importantly--understanding the policies that are being used. That complexity is something that Schreuders set out to reduce with FBAC. It remains to be seen if he has succeeded with that, but any such attempt is worthy of a look.
