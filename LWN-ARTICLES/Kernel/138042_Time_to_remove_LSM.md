---
title: "Time to remove LSM?"
url: https://lwn.net/Articles/138042/
date: "May 31, 2005"
category: "Security-Security modules"
author: 
---

[The first organized kernel summit](<http://lwn.net/2001/features/KernelSummit/>), held in 2001, included a presentation on the NSA Security-Enhanced Linux project. Linus's response at the time was that there were several projects out there trying to find the best way to harden Linux, and that he did not want to have to choose between them. Instead, he asked for the creation of a generic framework which would allow an arbitrary security module to be plugged into the system. The result, some time later, was the Linux Security Module framework; LSM provides a long list of hooks into kernel operations which allow a security module to veto any action which violates the rules it is implementing. 

The LSM patch ran into some difficulties on its way into the kernel, but it is now an established part of the internal API. So some developers were surprised recently when James Morris [suggested](<https://lwn.net/Articles/138045/>) that perhaps the time has come to remove the LSM framework. His [arguments](<https://lwn.net/Articles/138046/>) are simple: there is only one serious module using the LSM framework in the intended manner, while unrelated projects are trying to use it in inappropriate ways. 

In the years since LSM was included in the mainline kernel, SELinux has been the only significant module implemented and also included in the mainline kernel. So we have a generalized framework for one user, SELinux, which itself is a generalized framework.... 

It's dead code, an unnecessary abstraction layer between its one real user, SELinux, and the core kernel. 

James asks: rather than forcing SELinux to conform to a general-purpose API (of which it is the sole user), why not just wire SELinux directly into the kernel, get rid of LSM, and be done with it? 

SELinux is not truly the only security module out there, of course. The kernel includes a couple of other modules: a reimplementation of the capabilities mechanism and "root plug," a module which prevents processes from running as root unless a specific USB device is plugged in. There are out-of-tree modules, such as [the BSD securelevels patch](<https://lwn.net/Articles/60096/>) and [Trustees Linux](<https://lwn.net/Articles/111247/>). The Immunix (now Novell) [AppArmor](<http://immunix.org/products.php>) product includes a module which uses the LSM framework. AppArmor is a proprietary offering, but the security module portion of it is GPL-licensed (as is necessary, since the functions for loading security modules are exported GPL-only). 

There does not appear to be a groundswell of support for the idea of removing the LSM framework from the kernel at this time. That could change over time, however: increasingly, out-of-tree code is held to be irrelevant when decisions are made. If SELinux remains the only significant in-tree user of the LSM framework, LSM will look like useless baggage to more and more developers. If there are security modules out there which are reasonable alternatives to SELinux, their developers may want to think about getting them into the mainline sometime in the not-too-distant future.
