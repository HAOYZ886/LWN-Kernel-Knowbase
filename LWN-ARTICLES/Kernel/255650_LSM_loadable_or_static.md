---
title: "LSM: loadable or static?"
url: https://lwn.net/Articles/255650/
date: "October 24, 2007"
category: "Security-Security modules"
author: "By Jake Edge October 24, 2007"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
October 24, 2007

The ever-contentious Linux Security Modules (LSM) API is being debated once again on linux-kernel, not its removal, which Linus Torvalds came down firmly against, but whether it should allow security modules to be loaded dynamically. As part of 2.6.24, Torvalds merged a [patch to convert LSM into a static interface](<http://lwn.net/Articles/239962/>), but has indicated a willingness to revert it. The key sticking point is whether there are real security modules that require the ability to be runtime-loaded. 

A [complaint by Thomas Fricaccia](<http://lwn.net/Articles/255640/>) about the change caused Torvalds to put out a call for folks using module loading with their LSM code. The patch could be reverted if there are "real-world" uses for that ability. Torvalds again [questions the sanity of security developers](<http://lwn.net/Articles/255641/>), but is clearly looking for someone to step up: 

I'd like to note that I asked people who were actually affected, and had examples of their real-world use to step forward and explain their use, and that I explicitly mentioned that this is something we can easily re-visit. 

Jan Engelhardt responded with information about his [MultiAdmin](<http://jengelh.hopto.org/p/multiadm/>) module, which allows multiple root users on a system, each with their own UID. This allows separate tracking of file ownership, resource usage and the like for each administrator. MultiAdmin also allows for the creation of sub-administrators who can perform some root activities for processes and files owned by a subset of users. The use case he cites is for professors being allowed to administer their students' accounts without getting full root privileges. 

James Morris, who proposed the static LSM change, responded that MultiAdmin seemed to qualify as a real-world use under Torvalds's criteria. Though it is not clear that MultiAdmin _requires_ a loadable interface, it does use it. The venerable [root_plug security module](<http://www.linuxjournal.com/article/6279>) - which only allows root processes to start if a particular USB device is plugged in - also implements loading and unloading. In both cases, configuration _could_ be done via `sysfs` parameters with an enable flag to turn them on or off.

To some extent, for the examples offered so far, loading is a convenience for administrators, but the major users for unloading are developers. Crispin Cowan sums it up: 

Why would you want to dynamically unload a module: because it is convenient for debugging. Ok, so it is unsafe, and sometimes wedges your kernel, which sometimes forces you to reboot. With this patch in place, it forces you to *always* reboot when you want to try a hack to the module. 

Other justifications for leaving the LSM loadable interface in the kernel have been less compelling. It is hard to imagine that the US Sarbanes-Oxley regulation would allow loading security modules into a running kernel, but not allow the kernel to be rebuilt as Fricaccia [suggested](<https://lwn.net/Articles/255665/>). Inserting proprietary security modules that are provided from the vendor in a binary-only form seems foolhardy - this kind of potential abuse is the kind of hole Morris's patch was meant to close - but could be seen as a reason to allow LSM loading. 

A compromise may have been found in a [patch posted by Arjan van de Ven](<https://lwn.net/Articles/255666/>), which converts LSM to be either static or loadable depending on a compile-time kernel option. A consensus seems to be building that this is a reasonable approach, allowing distributions and users to decide for themselves whether they will allow security modules to be loaded. As of this writing, Torvalds has not weighed back in with a decision and the newly released 2.6.24-rc1 kernel has the static patch. 

Dynamic loading of security modules is a potential source of problems - what better place for a rootkit to hide? - but there are valid reasons that someone might want to use it. Linux strives to be open to many uses, including some that the kernel hackers might find distasteful; dynamic security modules would seem to be one of those uses.
