---
title: Progress in security module stacking
url: https://lwn.net/Articles/635771/
date: "March 11, 2015"
category: "Security-Security modules"
author: "By Jake Edge March 11, 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
March 11, 2015

It would seem that a long-running saga in kernel development may be coming to a close. Stacking (also composing or chaining) of Linux Security Modules (LSMs) has been discussed, debated, and developed in kernel security circles for many years; we have [looked](<https://lwn.net/Articles/315974/>) at the issue from a number of angles starting in 2009 (and [here](<https://lwn.net/Articles/316940/>)), but [patches](<https://lwn.net/Articles/109713/>) go back to at least 2004. After multiple fits and starts, it looks like something might finally make its way into the mainline kernel. 

In a nutshell, the problem is that any security enhancements that are suggested for the kernel are inevitably [pushed](<https://lwn.net/Articles/393008/>) toward the LSM API. But there can only be one LSM active in a given kernel instance and most distributions already have that slot filled. Linux capabilities would logically be implemented in an LSM, but that would conflict with any other module that was loaded. To get around that problem, capabilities have been hardwired into each LSM, so that the capability checks are done as needed by those modules. The [Yama LSM](<https://www.kernel.org/doc/Documentation/security/Yama.txt>) has also been manually stacked, if it is configured into the kernel, by calling its four hooks before the hooks from the active LSM are called. These are _ad hoc_ solutions that cannot really be used for additional modules that might need to all be active, so a better way has been sought. 

The last time we [looked in on the issue](<https://lwn.net/Articles/569861/>) was after the 2013 Linux Security Summit (LSS). Smack creator Casey Schaufler, who has been the most recent one to push stacking, presented his solution to attendees; he was looking for feedback on his approach. Schaufler's [proposal](<https://lwn.net/Articles/560802/>) was a complex solution that attempted to solve "all" of the stacking problems at once. In particular, it allowed using more than one of the LSMs that provide a full security model (the so-called "monolithic" LSMs: SELinux, Smack, TOMOYO, and AppArmor), which is a bit hard to justify in some eyes. For most, the pressing need for stacking is to support several single-purpose LSMs atop one of those monolithic security models, much like is done with Yama. 

In addition, Schaufler's patches tried to handle network packet labeling for multiple LSMs (to the extent possible) and added to the user-space interface under `/proc/PID/attr`. Each active LSM would have a subdirectory under `attr` with its attributes, while one LSM, chosen through a configuration option, would present its attributes in the main `attr` directory. These additions also added complexity, so the consensus that emerged from the 2013 LSS attendees was to go back to the basics. 

Schaufler has been working on that simplification. The 21st version of the patch set was [posted](<https://lwn.net/Articles/636056/>) on March 9, though the changes in this round are mostly just tweaks. The [previous version](<https://lwn.net/Articles/634622/>) picked up an ack from Yama developer Kees Cook, was tested by SELinux developer Stephen Smalley, and got a ""this version looks almost perfect"" from TOMOYO developer Tetsuo Handa. It looks like it could get into security maintainer James Morris's branch targeting the -next tree, which might mean we will see it in 4.1. 

The approach this time is a return to a much simpler world. Gone are the thoughts of stacking more than one monolithic LSM; this proposal creates a mechanism to stack the LSM hooks and to consult them when trying to decide on access requests. The interface for a given LSM used to be a `struct security_operations` that was filled out with pointers for each of the hooks to be called when making access decisions. That has been replaced with a union (`security_list_options`) that can hold a pointer to each of the different hook functions. That union is meant to allow for a single list type that can hold any of the hook functions, but still provide type checking. 

Instead of filling in the sparse `security_operations` structure, LSMs now initialize an array that contains each of their hooks. That gets handed off to the `security_add_hooks()` function that adds the hooks to the lists for each hook that the LSM infrastructure maintains internally. Those lists are initialized with the capabilities hooks; Yama hooks are then added if that LSM is configured for the kernel. For the rest of the LSMs, all of which are monolithic, only one can be chosen at boot time to have its hooks added to the list. 

When an access decision needs to be made, the hooks are called in the order that they were added. Unlike some previous iterations, the access checking will terminate when any of the hooks on the list denies access. If none do, then the access is allowed. 

That puts all of the machinery in place to provide stacking, but it doesn't allow choosing more than one of the monolithic LSMs on any given kernel boot. Multiple monolithic LSMs can be configured into the kernel, and one be specified as the default, but that can be overridden with the `security=` kernel boot parameter. New LSMs could be added to the kernel code, like Yama has been, but those will presumably be configured into the kernel at build time. 

Currently, Yama is the only smaller LSM in the tree and it is chosen (or not) at build time; the others are either not optional (capabilities) or can only have a single chosen representative added into the hook list at kernel initialization time. Essentially, Schaufler's patches avoid multiple monolithic modules that are active in a given boot by not providing a mechanism to choose more than one. That avoids the conflicts and complexity that earlier attempts had run aground on. As he noted: 

The stacking of modules that use the security blob pointers cred->security, inode->i_security, etc has not been addressed. That is future work with a delightful set of issues. 

Another change that Schaufler has made is to split the `security.h` header file for LSMs in two: one for the internal, common LSM-handling mechanism (which stays in `security.h`) and one that defines the hooks and macros that will be used by LSMs (which is contained in the new `lsm_hooks.h` file). While that change is large in terms of lines of code, it is largely janitorial, but it will make the interface boundaries clearer. 

If Schaufler's patches make it into the mainline, that may spur some of the smaller out-of-tree LSMs to "come in from the cold" and get submitted to the mainline. It may also help to remove the "single LSM" barrier that crops up when new security protections are proposed for the kernel. Providing a mechanism to support these kinds of protections, while steering clear of core kernel code, could lead to more of those protections in the mainline and, eventually, available in distributions. It will be interesting to see what _that_ leads.
