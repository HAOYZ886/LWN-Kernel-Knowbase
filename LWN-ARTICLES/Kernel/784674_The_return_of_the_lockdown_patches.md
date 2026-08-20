---
title: The return of the lockdown patches
url: https://lwn.net/Articles/784674/
date: "April 3, 2019"
category: "Security-UEFI secure boot"
author: "By Jake Edge April 3, 2019"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
April 3, 2019

It's been a year since we [looked](<https://lwn.net/Articles/750730/>) in on the kernel lockdown patches; that's because things have been fairly quiet on that front since there was a [loud and discordant dispute](<https://lwn.net/Articles/751061/>) about them back then. But Matthew Garrett has been posting new versions over the last two months; it would seem that the changes that have been made might be enough to tamp down the flames and, perhaps, even allow them to be merged into the mainline. 

The idea behind kernel lockdown is to supplement secure boot mechanisms to limit the ability of the root user to cause unverified, potentially malicious code to be run. The most obvious way to do that is to use the kexec subsystem to run a new kernel that has not been vetted by the secure boot machinery, but there are lots of other ways that root can circumvent the intent of (some) secure boot users. While the support for [UEFI secure boot](<https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface#SECURE-BOOT>) has been in the kernel for years, providing a way to [restrict the root user](<https://lwn.net/Articles/514985/>) after that point has always run aground. 

#### A renewed push

The latest round began with a [pull request from Garrett](<https://lwn.net/ml/linux-kernel/CACdnJuvW47m3JvEcuEX1bsr+L2Ht9LDn_iCuPbHLOoaohOFW4Q@mail.gmail.com/>) at the end of February. He noted that he had taken over shepherding the patch set from David Howells, who is ""low on cycles at the moment"". There were just a few changes from the previous version that caused the ruckus a year ago. 

The main change was to remove the tie-in between secure boot and lockdown mode. The main complaint that Linus Torvalds and Andy Lutomirski had a year ago was about that linkage; they felt that it was unreasonable to force those using secure boot into having a locked-down kernel—and vice versa. At a minimum, kernel developers might well want the flexibility to have one without the other. Changing the fundamental behavior of the kernel based on a BIOS setting that might not be under the control of the user was also seen as highly problematic. 

Beyond that big ticket item, there were two other changes. A `CONFIG_KERNEL_LOCK_DOWN_FORCE` option was added that will build a kernel that always enforces lockdown. Integration with the [Integrity Measurement Architecture](<https://sourceforge.net/p/linux-ima/wiki/Home/>) (IMA) was also dropped, though IMA maintainer Mimi Zohar [questioned that plan](<https://lwn.net/ml/linux-kernel/1551392438.10911.227.camel@linux.ibm.com/>). There were enough comments that needed addressing to cause Garrett to send a [second pull request](<https://lwn.net/ml/linux-kernel/20190306235913.6631-1-matthewgarrett@google.com/>) to security maintainer James Morris in early March. 

Zohar was still [unhappy with the (lack of) IMA integration](<https://lwn.net/ml/linux-kernel/1551930990.31706.279.camel@linux.ibm.com/>), however. Garrett [worked on a solution](<https://lwn.net/ml/linux-kernel/CACdnJuszfo4gTUD88p+8jeHU5e3hSdp+GEr8_=F1AMThPb7yWg@mail.gmail.com/>) to that, which showed up as a [patch](<https://lwn.net/ml/linux-kernel/20190325220954.29054-28-matthewgarrett@google.com/>) in a third [pull request on March 25](<https://lwn.net/ml/linux-kernel/20190325220954.29054-1-matthewgarrett@google.com/>). The patch will use the IMA architecture-specific mechanism to verify a kernel image before allowing it to be booted via kexec: 

Systems in lockdown mode should block the kexec of untrusted kernels. For x86 and ARM we can ensure that a kernel is trustworthy by validating a PE [[Portable Executable](<https://en.wikipedia.org/wiki/Portable_Executable>)] signature, but this isn't possible on other architectures. On those platforms we can use IMA digital signatures instead. 

A [patch](<https://lwn.net/ml/linux-kernel/20190325220954.29054-24-matthewgarrett@google.com/>) that disables the use of the [`bpf()` system call](<http://man7.org/linux/man-pages/man2/bpf.2.html>) in locked-down kernels was also discussed. There are some BPF functions that can read kernel memory, which would allow BPF programs to extract private keys (e.g. the hibernation image signing key) and to alter kernel memory, so the patch simply disabled `bpf()` entirely. But, given the ever-increasing use of BPF in the kernel, that was seen as a draconian restriction by some. Jordan Glover [pointed out](<https://lwn.net/ml/linux-kernel/dPp-BlyzEqZTTJfOrFzxSsFZj1eDOH4k4Lx7Qc0DGBdsRXPJHP-K5866dzAoZTvw4-GsBRHc4bfkL2zrajh8-KXWeR2B0pLIy0SIjK0-K5c=@protonmail.ch/>) that disabling the system call would break some systemd functionality, making locked-down systems _less_ secure. 

Disabling BPF was one of the problems that Lutomirski [saw](<https://lwn.net/ml/linux-kernel/CALCETrU=EqqRwyQGBT0=_N9-E_pBM53=z+WJrd9OzV7g6xJa0A@mail.gmail.com/>) with Garrett's approach to decoupling secure boot and lockdown mode. In particular, Lutomirski wanted to see three possible states for lockdown: 

Lockdown mode becomes three states, not a boolean. The states are: no lockdown, best-effort-to-protect-kernel-integrity, and best-effort-to-protect-kernel-secrecy-and-integrity. And this BPF mess illustrates why: most users will really strongly object to turning off BPF when they actually just want to protect kernel integrity. And as far as I know, things like Secure Boot policy will mostly care about integrity, not secrecy, and tracing and such should work on a normal locked-down kernel. So I think we need this knob. 

The [code](<https://lwn.net/ml/linux-kernel/20190325220954.29054-13-matthewgarrett@google.com/>) for disabling direct [model-specific register](<https://en.wikipedia.org/wiki/Model-specific_register>) (MSR) writes on x86 systems was also questioned. Writing to MSRs can ""lead to execution of arbitrary code in kernel mode"", which is why it should be disabled for locked-down kernels. At the behest of Alan Cox, log messages were added to someday facilitate a whitelist of allowed MSR writes, but Thomas Gleixner was [not a fan](<https://lwn.net/ml/linux-kernel/alpine.DEB.2.21.1903260003040.1789@nanos.tec.linutronix.de/>): 

Maintaining a whitelist for this is a horrible idea as you will get a gazillion of excuses why access to a particular MSR is sane. And I'm neither interested in these discussions nor interested in adding the whitelist to this trainwreck. 

Gleixner would much rather see direct access to `/dev/msr` go away entirely: ""The right thing to do is to provide sane interfaces and that's where we are moving to."" 

Another [complaint](<https://lwn.net/ml/linux-kernel/20190326003137.GB5112@kroah.com/>) came from Greg Kroah-Hartman, who said that the [heuristic-based patch](<https://lwn.net/ml/linux-kernel/20190325220954.29054-26-matthewgarrett@google.com/>) that restricted debugfs operations for locked-down kernels should instead simply disable debugfs completely. Garrett [noted](<https://lwn.net/ml/linux-kernel/CACdnJus6a_7VUR27K7dJE037b3TPwDNMh1AyAOptHepB18vjxw@mail.gmail.com/>) that previous attempts to do so had resulted in ""strong pushback from various maintainers"", but Kroah-Hartman [said](<https://lwn.net/ml/linux-kernel/20190326004351.GA26712@kroah.com/>) he was willing to handle any of those that come along. 

#### Version 31

He got a chance to do just that after Garrett [posted](<https://lwn.net/ml/linux-kernel/20190326182742.16950-1-matthewgarrett@google.com/>) version 31 of the patch set. It addressed the complaints, starting with lockdown state: 

Based on Andy's feedback, lockdown is now a tristate and can be made stricter at runtime. The states are "none", "integrity" and "confidentiality". "none" results in no behavioural change, "integrity" enables features that prevent untrusted code from being run in ring 0, and "confidentiality" is a superset of "integrity" that also disables features that may be used to extract secret information from the kernel at runtime. 

[...] 

In the general case, I'd expect distributions to opt for nothing stricter than "integrity" - "confidentiality" seems more suitable for more special-case scenarios. 

Beyond that, he removed the logging from the MSR-disabling code and disabled opening files in debugfs when in integrity mode. Perhaps predictably, that latter part led to a [complaint](<https://lwn.net/ml/linux-kernel/CALCETrXDF2c2-gnOyNTnEn-uPAZp0AaSB_PB8e2y8jf7zo=Cyg@mail.gmail.com/>). Lutomirski said that reading debugfs files should still be allowed for integrity mode. Kroah-Hartman, who doesn't think much of the lockdown idea in general, [said](<https://lwn.net/ml/linux-kernel/20190327053342.GA17484@kroah.com/>) that there are legitimate worries about what kinds of information debugfs provides: 

Reading a debugfs file can expose loads of things that can help take over a kernel, or at least make it easier. Pointer addresses, internal system state, loads of other fun things. And before 4.14 or so, it was pretty trivial to use it to oops the kernel as well (not an issue here anymore, but people are right to be nervous). 

Personally, I think these are all just "confidentiality" type things, but who really knows given the wild-west nature of debugfs (which is as designed). And given that I think this patch series [is] just crazy anyway, I really don't care :) 

Garrett [seems amenable](<https://lwn.net/ml/linux-kernel/CACdnJus=zXgmNas+XVkWfe+ohAHEGMYLtfuYe0TH-eE=4p0oRw@mail.gmail.com/>) to changing integrity mode to use the previous scheme and to block all reads in confidentiality mode, but doesn't want to ""spend another release cycle arguing about it"". That previous scheme would only allow opening "safe" debugfs files for read: those with a 00444 mode and lacking `.ioctl()` and `.mmap()` methods. 

Overall, the comments seem to be fairly minor problems that can be—have been—addressed easily. While some don't buy the whole idea behind lockdown, and there will always be ways around any of its restrictions due to bugs if nothing else, it is something that some kernel users want. Distributions have been shipping with some form of lockdown for quite some time, so it is pretty hard to argue that it is completely useless. 

But, of course, the elephant in the room is Torvalds. He has not commented on any of the recent postings. One might guess that most of his concerns were addressed by the decoupling of secure boot and lockdown mode, but that remains to be seen. Morris has not yet said he will merge the lockdown patches either, which would also seem to be a prerequisite. Reducing out-of-tree patches that distributions feel they need to carry is a good goal, though, so one way or another it seems likely that lockdown will get merged before too long.
