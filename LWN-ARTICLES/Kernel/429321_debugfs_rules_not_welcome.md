---
title: "debugfs: rules not welcome"
url: https://lwn.net/Articles/429321/
date: "February 22, 2011"
category: "Development model-Security issues; Security-Vulnerabilities"
author: "By Jonathan Corbet February 22, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 22, 2011

The kernel's debugfs filesystem is meant to be a place where kernel developers can place any information which seems to be of value to somebody. Unlike the other kernel virtual filesystems (`/proc`, `/sys`), debugfs has an explicit "no rules" rule. Anything developers want to put there is fair game, without regard for taste, (hypothetically) ABI stability, or perceived usefulness. "No rules" does not extend as far as compromising the security of the system, though, which has led to an attempt to lock debugfs down. 

Eugene Teo recently posted [a request for CVE numbers](<https://lwn.net/Articles/429323/>) for 20 separate vulnerabilities involving world-writable files in debugfs and sysfs. Some of the debugfs vulnerabilities would seemingly allow any local user to write arbitrary values into device registers - a situation from which little good can be expected to emerge. Expect yet another set of kernel updates in the near future as these holes are closed and fixes are made available to users. 

In response to these vulnerabilities, Kees Cook posted [a patch](<https://lwn.net/Articles/429326/>) which would cause debugfs to be mounted with root-only access permissions. That way, any future mistakes in debugfs would be inaccessible to nonprivileged users and, thus, would not be a new vulnerability in need of fixing. The patch was not received well; it looks suspiciously like a rule in a land where there are supposed to be no rules. Greg Kroah-Hartman [responded](<https://lwn.net/Articles/429327/>): 

It's just stupid mistakes being made here, don't try to lock down the whole filesystem for just a handful of bugs. 

Kees suggested that these mistakes could keep on happening, and that "no rules" might not be the best approach, but Alan Cox [responded](<https://lwn.net/Articles/429328/>): 

It's a debugging fs, it needs to be "no rules" other than the obvious "don't mount it on production systems" 

There is one little problem with the idea of not mounting debugfs on production systems, though: there is useful stuff in that filesystem. At the top of the list must certainly be the control files for perf and ftrace; most of our nice, new tracing infrastructure will not work without debugfs. There are also knobs for tweaking scheduler features, interfaces for the "usbmon" tool, interfaces used by Red Hat's kvm_stat tool, and so on. There is enough useful stuff in debugfs that is it can be found mounted well outside of kernel debugging environments; it has reached the point that Greg [challenges](<https://lwn.net/Articles/429332/>) the idea that debugfs should not be mounted on production systems: 

No, not true at all, the "enterprise" distros all mount debugfs for good reason on their systems. 

"No rules" and "mounted on enterprise systems" seems like a bad combination; it would be nice to make things more secure. A number of proposals have been floated to do that, including: 

  * Teach the checkpatch.pl tool to look for world-writable debugfs files and complain about them. This step has already been taken; the version of checkpatch.pl found in 2.6.38 will point out world-writable files in either debugfs or sysfs. 

  * Disallow world-writable files in debugfs. A patch has been posted to this effect; so far, there have been few comments to indicate whether such a restriction would look too much like a rule for debugfs or not. 

  * Move generally useful interfaces out of debugfs to a place with a bit less of a wild-west flavor, then leave debugfs unmounted on most systems. This is an idea which makes a lot of sense on the face of it, but it can also run into practical difficulties. Moving interfaces requires possibly cleaning them up, making a stronger commitment to ABI compatibility going forward and, importantly, breaking tools which depend on the current location of those interfaces. 

The last concern could be a show stopper; it could force developers to maintain both the old and new interfaces in parallel for some years. Many developers, faced with that sort of task, may just decide to leave the interface where it is. Debugfs is not supposed to have any ABI guarantees, but, as has [become clear](<https://lwn.net/Articles/309298/>) in the past, such a policy does not necessarily prevent the creation of an ABI which must be maintained going forward. 

So debugfs on production systems seems likely to be with us for some time. Given that, there is no alternative to making it more secure. The checkpatch.pl change is a good start, but it cannot take the place of proper code review. Reviewers have a tendency to skip over debugfs code, but, if that code is to run on important systems, that tendency must be fought. Debugfs code must uphold the security of the system just like any other kernel code.
