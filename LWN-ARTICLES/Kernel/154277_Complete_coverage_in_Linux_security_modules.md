---
title: Complete coverage in Linux security modules
url: https://lwn.net/Articles/154277/
date: "October 3, 2005"
category: "Modules-Security modules; Security-Security modules"
author: 
---

The Linux Security Module (LSM) framework is intended to allow security modules to lock down a system by inserting checks whenever the kernel is about to do something interesting. A security module hooks into those check points and, for each operation, convinces itself that the operation is allowed by the security policy currently in force. This approach can work well _if_ checks have been placed in all of the relevant locations. A missing check could open a door allowing a user-space process to do something which the site's policy would disallow. 

Kostik Belousov recently [noticed](<https://lwn.net/Articles/154278/>) this sort of problem in the 2.6 kernel: it seems that the `readv()` and `writev()` system calls ran without calling the associated LSM hook. The missing check means that a process which uses these calls (rather than `read()` or `write()`) could perform file I/O which was not subject to oversight by any security modules currently loaded in the system. The practical effect of this vulnerability is minimal: any security module worth its bits will have done its access checks when the file is opened, so the ability to do unchecked reads and writes should not open any gaping holes in the system. 

The more important point is how easily this sort of opening can come about. When the security modules patch was originally merged into the kernel, it included checks on `readv()` and `writev()`. But those system calls were later rewritten, and the LSM hooks fell by the wayside. This change [apparently happened around 2.5.47](<https://lwn.net/Articles/154282/>), but it only came to light now. 

Most kernel developers are only peripherally aware of the LSM system. Very few of them know how to code an LSM call, and the rules for the insertion of LSM checks are not particularly well documented. Code which is missing an LSM call still appears to work just fine in normal testing and use. The end result of all this is that it is trivially easy to omit an important check, or to delete one by accident. Such mistakes can then go unnoticed for years. 

Anybody who depends on a Linux security module (such as SELinux) is depending on comprehensive checking within the kernel. But, as has been demonstrated here, it is hard to feel sure that the LSM checks are, indeed comprehensive. There are many code paths through the kernel. When a relatively simple system call can go unprotected for so long, how secure do we feel about the more complex paths? It would seem that a thorough audit is called for. An automated audit might even be better; it may well be possible to adapt a tool like `sparse` to detect unchecked paths through the kernel. Some work in this area could do a lot to increase the level of trust which can be placed in LSM-based modules.
