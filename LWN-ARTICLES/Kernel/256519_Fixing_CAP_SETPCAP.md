---
title: Fixing CAP_SETPCAP
url: https://lwn.net/Articles/256519/
date: "October 31, 2007"
category: Capabilities
author: "By Jake Edge October 31, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
October 31, 2007

Linux capabilities have been around for almost ten years now - they were originally merged into a 2.1 kernel - but they haven't gotten a lot of use in that time. One pretty basic missing feature, support for [associating capabilities with files](<http://lwn.net/Articles/211883/>), has been merged for 2.6.24. This allows a longstanding hack, which redefines the proper usage of CAP_SETPCAP, to be fixed; this too has been [merged into 2.6.24](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commit;h=72c2d5823fc7be799a12184974c3bdc57acea3c4>).

A bit of review is probably in order. Capabilities are a way to separate individual privileges that are normally all granted to the root user. There are currently 31 different capabilities defined (in `linux/capability.h`), but there are [efforts underway to allow for expansion](<https://lwn.net/Articles/256541/>). The idea is that a program should be able to set the system time, for example, without needing the entire set of privileges that come with a `setuid(0)` program. 

Capabilities originally came from a proposed POSIX standard that was eventually not adopted, but, in the meantime, got included into Linux. The feature has languished since, for a number of reasons, but perhaps the largest was that there was no way to associate executable programs with a set of capability bits. Now that capability bits can be stored in the extended attributes of files, the process can get the proper capabilities when the program is invoked. Standard UNIX permissions still apply - users can only execute programs they have an `x` bit for. 

In order to use capabilities at all, prior to being able to store them with files, a method was needed to set the capabilities of a running process. The `CAP_SETPCAP` capability was co-opted for this purpose. A process with this capability, which, in practice, meant root processes could set the capabilities of another process. If that process was meant to be able to do the same - something that needs to be carefully considered - it could get the `CAP_SETPCAP` bit as well. 

This could really only be used to add capabilities to long running processes that were not run as root (which has all of the capabilities), or to remove some capabilities from daemons run as root. Other schemes using `setuid` wrappers for utility programs that needed some privileges could also be imagined, but distributions or tools that use capabilities are not widespread. 

`CAP_SETPCAP` was never meant to have this behavior, so the recent patch restores it its original meaning. As odd as it might seem at first, `CAP_SETPCAP` is only meant to allow changes to a process's own capabilities; in fact, with this patch applied, there is no way for a process to change a running process's capabilities. That is probably the biggest user-visible change. 

Capabilities are not a single set of bits, but are instead, three sets of bits representing the effective, permitted, and inheritable capabilities of a process. Files, similarly, have three capability sets which are combined with those of the process executing the file using the "capability rules" (described in the patch and in an [LWN article from a year ago](<http://lwn.net/Articles/199004/>)) to determine the three sets for the process created. 

For processes, the effective set contains those capabilities currently enabled - a process might drop some that it is allowed once it has performed the corresponding privileged operation - while the permitted set is a superset of the effective set, including all capabilities allowed to that process. The inheritable set are those that are passed on to a new program started by an `exec()` call, which is where the new `CAP_SETPCAP` comes into play; a process with this capability can change its inheritable set to include any capability, including those that are not in their permitted set. 

This allows processes to bestow privileges that they do not possess upon their children, which provides for some interesting uses. It helps further partition privileges by not requiring a process to have a particular capability simply to pass it on to children. The example provided in the patch illustrates this nicely: the `login` program does not require many privileges, but through some policy mechanism (pam_cap for example) could allow certain users to have extra capabilities. Because the login process does not itself possess those extra capabilities, this could limit the damage an exploit of `login` could do. 

It is unclear whether these recent additions to the capability feature set will result in more capability users. There is a lot of work in the kernel security space right now as kernel hackers and security folks try to come up with sensible security solutions for Linux. The complexity of SELinux, along with the fact that many administrators disable it rather than try to figure it out, seems to have the community casting about for other solutions. It is possible that capabilities might be a part of another solution, though its complexities are far from trivial. Though most of the major distributions have already made their security model choice, a capabilities-based distribution would be interesting to see; it might make a nice project for a smaller, up-and-coming, distribution to try.
