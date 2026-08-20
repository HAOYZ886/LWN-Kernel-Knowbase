---
title: SMACK meets the One True Security Module
url: https://lwn.net/Articles/252562/
date: "October 2, 2007"
category: "Security-Security modules"
author: "By Jonathan Corbet October 2, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 2, 2007

The Simplified Mandatory Access Control Kernel is a security module designed to harden Linux systems through the addition of mandatory access control policies; it was [covered here](<http://lwn.net/Articles/244531/>) last August. Like SELinux, SMACK works by attaching labels to processes, files, and other system objects and implementing rules describing what kinds of access are allowed by specific combinations of labels. Unlike SELinux, though, SMACK was designed specifically for simplicity of administration. 

The posting of [version 3 of the SMACK patch](<http://lwn.net/Articles/252378/>) inspired a certain amount of discussion. [Andrew Morton's take](<https://lwn.net/Articles/252568/>) led things off: 

I don't know enough about security even to be dangerous. I went back and reviewed the August thread from your version 1 submission and the message I take away is that the code has been well-received and looks good when considered on its own merits, but selinux could probably be configured to do something sufficiently similar. 

I'd have trouble declaring that "but" to be a reason to not merge smack. I'm more thinking "let's merge it and see if people use it". 

Andrew concluded by suggesting that the SMACK patch should be merged for 2.6.24. There was some discussion about specific pieces of the patch (some developers are more impressed by CIPSO network labels than others), but there does not seem to be a whole lot of opposition to merging the SMACK security module. It is generally seen as being well-written code which makes sense from a security point of view. 

There was one predictable exception, though: whenever a new security module is considered for merging the SELinux developers tend to oppose it. This time, James Morris voiced [a more general complaint](<https://lwn.net/Articles/252569/>): SMACK would become the second in-kernel user of the Linux security module (LSM) framework. That, in turn, would make it almost impossible to remove LSM from the kernel. James claims: 

If LSM remains, security will never be a first class citizen of the kernel. Application developers will see multiple security schemes, and either burn themselves trying to support them, or more likely, ignore them. Core kernel developers will continue to not have enough information to understand what the LSM hooks in their code are supposed to be doing, leading to long term maintenance issues and potential security problems. 

The SELinux programmers, it seems, would rather that SELinux became the one security framework supported by Linux, with all development effort going into making SELinux better. 

The problem, of course, is that, even after a few years with only SELinux in the kernel, there has not been a glut of application developers working to support it. The vision of each application shipping with an SELinux policy file has not come to be. Some progress has been made in the creation of higher-level tools for the management of SELinux policies, and the Fedora and Red Hat developers have come a long way toward making a general-purpose distribution work with a limited set of SELinux policies, but SELinux simply remains too complex for most people to deal with. SELinux may work out of the box on an RHEL system, but as soon as the system administrator runs into something which breaks, chances are that SELinux will be turned off. 

The SELinux developers would presumably argue that the way to address these problems is to focus on a single security framework within the kernel. Rather than create competing, simplified frameworks, it would be better to implement approaches like AppArmor or SMACK within SELinux and, in the process, make SELinux better. Those developers [argue](<https://lwn.net/Articles/252587/>) that pluggable security modules, like pluggable schedulers, succeed only in splitting development effort and preventing the creation of a truly general solution: 

Do you really want to encourage people to roll their own security module rather than working toward a common security architecture and a single balanced solution (which doesn't necessarily mean SELinux, mind you, but certainly could draw from parts of it)? As with pluggable schedulers, the LSM approach prevents cross pollination and forces users to make poor choices. 

The answering argument is that there are many security environments and differing sets of user needs; there is no sign that a single security approach can ever work in all situations. And, even if such a unified approach were possible, getting security developers to agree on it would still be a major challenge. Ted Ts'o [writes](<https://lwn.net/Articles/252588/>): 

The real problem with security is that there are no "hard numbers", as Linus puts it. Instead, there are different sets of requirements in terms of what is a valid threat model --- which will very depending on the environment and how the servers are deployed, and what the capabilities are of the adversary trying to penetrate said system --- and how end users are willing to compromise between security and convenience. 

The LSM architecture was a result of [the very first kernel summit](<http://lwn.net/2001/features/KernelSummit/>), where Linus stated that he had no wish to choose between the various security ideas in circulation and, instead, wanted the kernel to be able to support all of them. He has not changed his mind since; he still doesn't see any imminent consensus on security approaches, and still wants Linux to be flexible in this regard. So [his pronouncement](<https://lwn.net/Articles/252589/>) was: 

So LSM stays in. No ifs, buts, maybes or anything else. 

When I see the security people making sane arguments and agreeing on something, that will change. Quite frankly, I expect hell to freeze over before that happens, and pigs will be nesting in trees. But hey, I can hope. 

So it looks like the way is clear for a merger of SMACK in 2.6.24. Once that happens, it would not be surprising to see another push made by the developers of security modules like AppArmor and TOMOYO Linux; in fact, the [TOMOYO Linux patches](<https://lwn.net/Articles/252652/>) have just been reposted. In the end, SELinux will, despite the apparent wishes of its developers, have to compete with other security approaches for attention from developers, distributors, and users. There will be no One True Security Module for Linux in the foreseeable future.
