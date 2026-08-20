---
title: "The return of loadable security modules?"
url: https://lwn.net/Articles/526983/
date: "November 28, 2012"
category: "Modules-Security modules; Security-Security modules"
author: "By Jake Edge November 28, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
November 28, 2012

The idea behind the Linux Security Module (LSM) interface was initially discussed as part of the "NSA Linux" session at the [first Kernel Summit](<http://lwn.net/2001/features/KernelSummit/>) back in 2001. The intent was to avoid wiring a particular security solution into the kernel; instead, multiple approaches to security could be built on top of a common kernel API. Originally, as the name implies, the solutions were built as loadable kernel modules, but eventually the "M" in LSM became just a historical artifact as the API was no longer exported to modules (essentially requiring security "modules" to be statically linked into the kernel). But it's possible that may all change again with a recent [patch to bring back loadable LSMs](<https://lwn.net/Articles/525729/>). 

#### Some history

A bit of history is probably in order. The LSM API came about specifically because Linus Torvalds didn't want to have to choose between a number of competing access control mechanisms for the kernel. Instead, LSM would provide a way for any of those mechanisms to hook into the kernel and deny access to various kinds of resources (files, devices, tasks, inodes, etc.) based on the security model being implemented. Initially, the LSMs would be implemented as kernel modules that could be loaded at runtime and, in some cases, unloaded. 

The LSM interface was released as part of the 2.5 development kernel series in 2002, and was part of the first 2.6 release in December 2003. For several years after that, there was only one in-tree user of the interface: SELinux. That led to a 2005 [suggestion](<https://lwn.net/Articles/138042/>) to remove the LSM API entirely, effectively just calling SELinux directly. That would turn SELinux into the "one true security solution" for Linux. In 2006, James Morris [proposed a patch](<https://lwn.net/Articles/180194/>) to move LSM to the "feature removal" list, scheduled for the 2.6.18 kernel, which was roughly two months out at that point. 

But, along came [Smack](<https://lwn.net/Articles/244531/>), which implemented a "simplified" Mandatory Access Control (MAC) scheme for the kernel. It also used the LSM interface, so, to a certain extent, the [decision on whether to merge it](<https://lwn.net/Articles/252562/>) hinged on the future of LSM. In October 2007, Torvalds [clearly stated](<https://lwn.net/Articles/252589/>) his intention to keep LSM in the kernel, thus paving the way for Smack to be merged. 

At more or less the same time Smack was merged, another change to LSM was made. First [discussed](<https://lwn.net/Articles/239962/>) in mid-2007, Torvalds merged a patch for the 2.6.24 kernel that [switched LSM to a static interface](<https://lwn.net/Articles/255650/>) so that security "modules" needed to be built into the kernel. One could still choose which security module to use with kernel command-line parameters, but dynamic security module loading would no longer be allowed. 

There were a number of reasons behind the switch. For one thing, unloading modules was always messy (or impossible), partly because keeping a coherent security state through that process is difficult. In addition, the LSM API is very intrusive, allowing modules to hook nearly any kernel operation, which can be (and was) abused. While the LSM symbols were exported as GPL-only, that didn't stop some proprietary modules from abusing the interface. There were also free software modules that used the interface for non-security purposes (e.g. [the realtime "security" module](<https://lwn.net/Articles/118785/>)). Those kinds of problems could also be used as arguments against having the LSM API at all, but since Torvalds had already put his foot down on that particular question, removing the ability to load LSMs was seen as a reasonable alternative. 

At the time that Torvalds merged the patch that made that switch, he asked for "real world" users of loadable security modules to step forward. There were a few examples of out-of-tree LSMs that were loadable (and, possibly, unloadable), but none that actually seemed to _require_ that ability. The main users of the feature were LSM developers, who might routinely load and unload their LSM during development. 

The next few years saw the merging of Smack (2.6.25), TOMOYO (2.6.30), and AppArmor (2.6.38). The latter had been long out of tree; its existence was part of the reason that the LSM interface came about in the first place. There have also been periodic attempts to get smaller, single-purpose security changes into the kernel over the years, but those were always pointed to the LSM interface. There is a problem with that particular suggestion, though, as only one LSM can be active at a time. Most distributions already have their one security module "slot" filled up. Red Hat and Fedora use SELinux, Ubuntu uses AppArmor, while SUSE and openSUSE have both AppArmor and SELinux available. Adding a specialized LSM for additional security protections is generally not possible without removing or disabling the distribution-supplied security solution. 

#### Proposed LSM changes

That "one LSM at a time" problem has led to persistent (if intermittent) calls for ways to stack or chain LSMs. Smack developer Casey Schaufler is the most recent to [propose a stacking solution](<https://lwn.net/Articles/518345/>). His patch set has been steadily reviewed on the LSM mailing list since it was first posted in September; it is now up to [version 8](<https://lwn.net/Articles/525960/>). That particular version came with an interesting caveat: 

I have not tried to reintroduce LSMs as loadable modules, in spite of the vigor with which it has been requested. I see that as work for another day, and a [separate] battle to fight. 

Those requests came from the developer of the TOMOYO LSM, Tetsuo Handa. In earlier discussion of Schaufler's stacking patch, Handa [advocated](<https://lwn.net/Articles/527227/>) a return to allowing loadable LSMs. In fact, he went further than that, proposing a set of patches that would restore the ability to load LSMs as well as converting TOMOYO to use that feature. 

Handa lists three reasons for making the change. To start with, any distribution that wants to allow its users to experiment with different LSMs must build all of those LSMs statically into the kernel. That will not only increase the size of the kernel, it will also increase the time it takes to load and boot the kernel. Most of that space (and time) would be completely wasted even for the users who are experimenting. All of that makes it less likely that distributions will actually build kernels that way. 

Beyond that, though, many distributions have their preferred LSM, so they don't build extra LSMs into their kernels. That leaves users to build their own kernels, which is generally unacceptable, particularly in enterprise settings. But even if there are other LSMs built into the kernel, it takes a reboot to enable them. Handa notes that he uses a loadable kernel module that implements TOMOYO (called AKARI) to diagnose problems in enterprise systems. In order to access the LSM symbols (which are no longer exported), AKARI must do some kind of runtime address resolution, perhaps using `/proc/kallsyms` or `System.map`. But, AKARI is something he can load into running systems when needed--unlike regular LSMs. 

One could argue that Handa's use of an LSM for system troubleshooting is a misuse of the interface, but the fact remains that changing LSMs currently requires a reboot. That problem potentially becomes more acute if LSM stacking is merged. One must decide pre-boot which LSMs to enable (and in what order they are consulted). Whatever else can be said, disallowing LSM loading reduces flexibility. 

Handa's third reason is a bit more philosophical: ""LSM is not the tool for thought control."" Essentially, he argues, disallowing LSM loading just makes dealing with LSMs harder for both users and developers. It also means that the more "minor" LSMs (e.g. TOMOYO and Smack) get less exposure because fewer users can actually try them. 

While there have been no comments on Handa's patches as yet, there have been expressions of support for loadable LSMs by some. Schaufler, for example, does not seem opposed necessarily. Kees Cook [agreed](<https://lwn.net/Articles/527228/>) with the need for loadable LSMs, though he was concerned that combining it with the LSM stacking patches would potentially block the progress for stacking. Morris, who authored the original patch to block loadable LSMs, has not yet spoken up one way or the other. 

Taking away the ability to load LSMs did not really change the picture for the kinds of abuses that were brought up at the time the change was made. Kernel modules can still abuse the interface, though it may take a bit more work. If binary modules were willing to ignore the GPL-only export of the LSM interface, they are probably willing to ferret out the addresses they need instead. Open source modules can do much the same. At the time of the switch to a static interface back in 2007, Torvalds seemed very open to reverting it if there were real users--perhaps he can still be convinced.
