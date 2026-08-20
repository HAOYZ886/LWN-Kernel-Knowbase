---
title: Preparing the kernel for UEFI secure boot
url: https://lwn.net/Articles/514985/
date: "September 6, 2012"
category: "Security-UEFI secure boot"
author: "By Jonathan Corbet September 6, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 6, 2012

UEFI secure boot is a much-discussed mechanism by which a system's firmware will refuse to run a bootloader that is not signed with a recognized key. Its stated purpose is to thwart boot-time malware; in the absence of boot-time checks, it is said, suitably privileged code could hide itself deeply within the system. In the real world, secure boot is also useful as a platform lockdown mechanism. It now seems that secure boot will not be used to lock down x86-based systems or to prevent them from running Linux; the story on the ARM architecture is less encouraging. But, even on x86, "running Linux" is not quite the same as running the Linux system you have now; we are now beginning to see what kinds of changes will be needed to fit Linux into the secure boot environment. 

The problem, simply put, is this: the objective of secure boot is to prevent the system from running any unsigned code in a privileged mode. So, if one boots a Linux system that, in turn, gives access to the machine to untrusted code, the entire purpose has been defeated. The consequences could hurt both locally (bad code could take control of the machine) and globally (the signing key used to boot Linux could be revoked), so it is an outcome that is worth avoiding. Doing so, however, requires placing limitations in the kernel so that not even root can circumvent the secure boot chain of trust. 

The form of those limitations can now be seen in Matthew Garrett's [secure boot support patch set](<https://lwn.net/Articles/514806/>). These patches may see some changes before finding their way into the mainline, but chances are that their overall form will not evolve that much. 

The first step is to add a new capability bit. Capabilities describe privileged operations that a given process can perform; they vary from `CAP_DAC_OVERRIDE` (able to override file permissions) to `CAP_NET_BIND_SERVICE` (can bind to a low-numbered TCP port) to `CAP_SYS_ADMIN` (can do a vast number of highly privileged things). The new capability, called `CAP_SECURE_FIRMWARE`, enables actions that are not allowed in the secure boot environment. Or, more to the point, its absence blocks actions that might otherwise enable the running of untrusted code. 

Naturally, the first thing reviewers complained about was the name. It describes actions that can be performed in the _absence_ of "secure firmware"; some reviewers have also disputed whether it has anything to do with security in the first place. So the capability will probably be renamed, though nobody has come up with an obvious replacement yet. 

Whatever it is eventually called, this capability will normally be available to privileged processes. If the kernel determines (by asking the firmware) that it has been booted in the secure mode, though, this capability will be removed from the bounding set before `init` is run; once a capability is removed from that set, no process can ever obtain it. Matthew's patch set also adds a boot-time parameter (`secureboot_enable=`) that can be used to simulate a secure boot on hardware that lacks that feature. 

In the secure boot world, processes lacking the new capability can no longer access I/O memory or x86 I/O ports. Either of those could be used convince a device to overwrite the running kernel with hostile code using DMA, compromising the system, so they cannot be allowed. One consequence is that graphics cards without kernel mode setting (KMS) support cannot be used; fortunately, the number of systems with (1) UEFI firmware and (2) non-KMS graphics is probably countable using an eight-bit signed value. Other user-space device drivers will be left out in the cold as well. Someday, Matthew says, it may be possible to enable I/O access on systems where the I/O memory management unit can enforce restrictions on the range of DMA operations, but, for now, all such access is denied. 

Similarly, all write access to `/dev/mem` and `/dev/kmem` must be disabled, even if the kernel configuration would otherwise allow such access. 

The strongest comments came in response to another limitation — the disabling of the `kexec()` system call. This call replaces the running kernel with a new kernel and boots the result without going through the system's firmware. It can be used for extra-fast reboots, though the most common use, arguably, is to boot a special kernel to create a crash dump after a system failure. Booting an arbitrary kernel obviously goes against the spirit of secure boot, so it cannot be allowed. 

Eric Biederman, in particular, complained about this limitation, [saying](<https://lwn.net/Articles/514996/>): 

This is Unix. In Unix we give root rope and let him hang himself or shoot himself in the foot (not that we encourage it). Why are we now implementing a security model where we don't trust root? 

Matthew [responded](<https://lwn.net/Articles/515121/>) that, in fact, we can't always trust root, and never have trusted it fully: 

Because historically we've found that root is also often someone who has determined a mechanism for running arbitrary code on your machine, rather than someone you trust. Root and the kernel aren't equivalent, otherwise root would just be able to turn off memory protection in their userspace processes. This patchset merely strengthens that existing dividing line. 

In this case, the proper solution would appear to be to allow `kexec()` to succeed if the target kernel has been properly signed. That support has not yet been implemented, though. It's apparently on the to-do list, but, as Matthew [said](<https://lwn.net/Articles/514997/>): ""We ship with the code we have, not the code we want."" 

One other important piece of the puzzle, of course, is module loading; if unsigned modules can be loaded into the kernel, the game is over. But, unlike `kexec()`, module loading cannot simply be turned off, so the implementation of some sort of signing mechanism cannot be put off. The module signing implementation is not part of Matthew's patch set, though; instead, David Howells has been [working on the problem](<https://lwn.net/Articles/470906/>) for some time now. This code has been delayed as the result of strong disagreements on how signing should be implemented; a solution was [worked out](<https://lwn.net/Articles/515007/>) at the 2012 Kernel Summit and this feature, in the form of [a new patch set](<https://lwn.net/Articles/514763/>) from Rusty Russell, should find its way into the mainline as soon as the 3.7 development cycle. 

The end result is that, by the time users have machines with UEFI secure boot capabilities, the kernel should be able to do its part. Whether users will like the result is another story. There is great value in knowing that the system is running the software you want it to be running, and many users will appreciate that. But others may find that the system is refusing to run the software they want; that is harder to appreciate. If things go well, the restrictions required by UEFI secure boot will come to be seen like other capability-based restrictions in Linux: occasionally obnoxious, but good for the long-term stability of the system and ultimately circumventable if need be.
