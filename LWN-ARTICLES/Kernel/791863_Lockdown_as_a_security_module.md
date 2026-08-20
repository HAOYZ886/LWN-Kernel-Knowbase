---
title: Lockdown as a security module
url: https://lwn.net/Articles/791863/
date: "June 24, 2019"
category: "Security-UEFI secure boot"
author: "By Jonathan Corbet June 24, 2019"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 24, 2019

Technologies like UEFI secure boot are intended to guarantee that a locked-down system is running the software intended by its owner (for a definition of "owner" as "whoever holds the signing key recognized by the firmware"). That guarantee is hard to uphold, though, if a program run on the system in question is able to modify the running kernel somehow. Thus, proponents of secure-boot technologies have been trying for years to provide the ability to lock down many types of kernel functionality on secure systems. The latest attempt posted by Matthew Garrett, at an eyebrow-raising [version 34](<https://lwn.net/ml/linux-kernel/20190622000358.19895-1-matthewgarrett@google.com/>), tries to address previous concerns by putting lockdown under the control of a Linux security module (LSM). 

The lockdown patches have a long and controversial history; LWN first [wrote about them](<https://lwn.net/Articles/514985/>) in 2012. Opposition has come at all kinds of levels; some developers see lockdown as a way of taking control of systems away from their owners, while others see it as ultimately useless security theater. There does appear to be some value, though, in making a system as resistant to compromise as possible, so these patches have persisted and are often shipped by distributors. Disagreement over more recent versions of the lockdown patch set were focused on details like [whether lockdown should be tied to the presence of secure boot](<https://lwn.net/Articles/751061/>) or [integration](<https://lwn.net/Articles/784674/>) with the integrity-measurement infrastructure. 

One outcome from the most recent discussion was a concern that the lockdown patches were wiring too much policy into the kernel itself. The kernel has long had a mechanism for pushing security-policy decisions out to user space — the security-module mechanism. So it arguably makes sense to move lockdown decision-making into an LSM; that is indeed what the more recent versions of the patch set do. 

First, though, there is the problem of initialization. LSMs exist to apply policies to actions taken by user space, so as long as the LSM infrastructure is running by the time user space starts, everything is fine. Lockdown, though, must act earlier: it needs to be able to block the action of certain types of command-line parameters and must be functional even before a security policy can be loaded. So the patch set starts by creating a new type of "early security module" that is initialized toward the beginning of the boot process. At this point, the module can't do much — even basic amenities like `kmalloc()` are not available — but it's enough to register its hooks and take control. 

Next, a new security hook is provided for LSMs to use: 

```
int (*locked_down)(enum lockdown_reason what);
```

This hook allows the LSM to decide whether kernel lockdown will prohibit a given action, described by `what`. Actions include loading unsigned modules (`LOCKDOWN_MODULE_SIGNATURE`), access to special files like `/dev/port` (`LOCKDOWN_DEV_MEM`), hibernating the system (`LOCKDOWN_HIBERNATION`), use of many perf functions (`LOCKDOWN_PERF`), kernel tracing (`LOCKDOWN_TRACEFS`), some BPF function (`LOCKDOWN_BPF_READ`), and many more. There are, it turns out, a lot of ways for root to compromise or extract information from the kernel; the lockdown patches do their best to close them all off. 

In the previous version of the patch, Andy Lutomirski [objected](<https://lwn.net/ml/linux-kernel/CALCETrVUwQP7roLnW6kFG80Cc5U6X_T6AW+BTAftLccYGp8+Ow@mail.gmail.com/>) to the provision of detailed actions to LSM `locked_down()` hooks. He argued, instead, for simply providing two values: "confidentiality" (for actions that may leak information from the kernel) and "integrity" (for those that could compromise the kernel). Doing anything else, he said, risks problems in the future if the meaning of any of those actions expands or changes. Garrett [thought the extra information could be useful](<https://lwn.net/ml/linux-kernel/CACdnJuv2sePuGBtTM6UL4S2k1UATcAk517o6vPx2EWF0Uxt8iw@mail.gmail.com/>), though, and was less worried about incompatible changes. So this feature remains in the current patch set. 

The patch set does maintain the distinction between confidentiality and integrity lockdown, though. The various `LOCKDOWN_*` constants are ordered so that the integrity-related actions have lower values than those related to confidentiality. All of the integrity-related actions have values less than `LOCKDOWN_INTEGRITY_MAX`; all actions of any type have lower values than `LOCKDOWN_CONFIDENTIALITY_MAX`. So a module that wants to distinguish between the two types of actions can do so using a simple comparison and without needing to understand each action specifically. 

That feature is used by the [static lockdown policy module](<https://lwn.net/ml/linux-kernel/20190622000358.19895-4-matthewgarrett@google.com/>) added by the patch set. While some users may want to put together complex LSM policies regarding lockdown, many others are likely to simply want to lock down everything and be done with it. The static policy module enables that in a number of ways, depending on how the kernel is configured. There is a `lockdown=` command-line parameter that can be set to either `integrity` or `confidentiality` to enable lockdown, which can also be turned on at run time via the `/sys/kernel/security/lockdown` sysfs knob. There are also, though, kernel configuration options to simply force either lockdown mode with no option for it to be disabled. 

Over the years, the lockdown patches have proved to be highly controversial. There are certainly still developers who are unconvinced by or opposed to the concept, but it seems that most of the objections have been addressed through years of patient reworking of the patch set. One can never be sure that 34 revisions will be enough until the code lands in the mainline kernel, but the odds seem relatively good that this long story is finally approaching an end.
