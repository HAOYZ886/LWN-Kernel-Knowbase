---
title: No fanotify for 2.6.36
url: https://lwn.net/Articles/409681/
date: "October 12, 2010"
category: fanotify
author: "By Jonathan Corbet October 12, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 12, 2010

The fanotify subsystem (originally "TALPA") was designed as a hook allowing anti-malware applications to intercept - and possibly block - file-oriented system calls. Getting this code into the mainline has been a long process, involving redefined requirements, reworking the low-level VFS notification code, and redesigning the user-space interface. After all that work, fanotify developer Eric Paris was able to get the code merged during the 2.6.36 merge window. Developers started using the interface to do interesting things; Lennart Poettering has mentioned, for example, using it to monitor file accesses to improve system bootstrap times. This long story, it seemed, was near an end. 

Along came Tvrtko Ursulin, who [pointed out](<https://lwn.net/Articles/409682/>) a problem with the fanotify system calls; he then followed up with [a second issue](<https://lwn.net/Articles/409683/>). It seems that the results of permission decisions were not always being handled correctly, and that the `fanotify_init()` system call had, somewhere along the way, lost the intended priority argument. The second issue, in particular, is serious because it affects the user-space ABI, which must be maintained indefinitely. 

Eric acknowledged the problems and started to ponder ways to get around them before the 2.6.36 release, but Alan Cox [advised](<https://lwn.net/Articles/409685/>) a more cautious approach: 

Given two chunks of "oh dear" last minute stuff would it be safer to simply punt and just pull the syscall/prototype itself (leaving the rest) for the release. That can go into the first pass of the next kernel tree, and if it the fixes and priority bits all work out may well then be tiny enough for -stable. 

Eric, not entirely pleased with the idea, carried on the discussion for a while. Eventually, though, he sent in [a patch](<https://lwn.net/Articles/409686/>) disabling the fanotify system calls: 

This feature can be added in an ABI compatible way in the next release (by using a number of bits in the flags field to carry the info) but it was suggested by Alan that maybe we should just hold off and do it in the next cycle, likely with an (new) explicit argument to the syscall. I don't like this approach best as I know people are already starting to use the current interface, but Alan is all wise and noone on list backed me up with just using what we have. I feel this is needlessly ripping the rug out from under people at the last minute, but if others think it needs to be a new argument it might be the best way forward. 

Linus took the patch, so, while the fanotify code will be present in the 2.6.36 release, it will not be accessible from user space. Whether the problems can be fixed in a way which is suitable for a 2.6.36.y stable release remains to be seen.
