---
title: UEFI secure boot kernel restrictions
url: https://lwn.net/Articles/523367/
date: "November 7, 2012"
category: "Security-UEFI secure boot"
author: "By Jake Edge November 7, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
November 7, 2012

The UEFI secure boot "problem" spans multiple levels in a Linux system. There are the actual "getting Linux to boot" issues, which have mostly been addressed by the [two signed bootloaders](<https://lwn.net/Articles/519618/>) that are available for distributions and users. Beyond that, though, are a set of kernel features that have the potential to subvert secure boot. Depending on one's perspective, those features either need to be configurable in the kernel--so some distributions can turn them off in their signed kernels--or they pose little risk beyond that of existing (but unknown) kernel bugs. As might be guessed, both sides of that argument can be heard in a recent linux-kernel thread. 

The root problem, so to speak, is that the root user on Linux systems is trusted by the kernel. That means root can make all sorts of temporary--or permanent--changes to the state of the system. Those changes include things like using `kexec()` to boot a different operating system, or writing a hibernation image to the swap partition for use in a resume. But both of those things could be used by an attacker to circumvent the protections that secure boot is meant to enforce. 

If, for example, a user were to boot using one of the Microsoft-signed Linux bootloaders--"shim" or "pre-bootloader"--into a kernel that didn't restrict root's powers, that kernel could arrange to execute a different kernel, perhaps one that has been compromised. Worse yet, from the perspective of those worried about Microsoft blacklisting bootloaders or keys, that second kernel could actually be a malicious version of Windows. So, a secure-boot-protected system would end up booting mal-Windows, which is precisely the scenario secure boot is supposed to prevent. 

If that occurs in the wild, various folks believe that Microsoft will blacklist the bootloader that was used in the attack. If that's the same bootloader used to boot Linux, new systems, as well as old systems that get the blacklist update, will no longer boot Linux. Matthew Garrett (at least) is concerned about that scenario, so he has [proposed kernel changes](<https://lwn.net/Articles/514985/>) that would prevent suitably configured kernels from using `kexec()` among a handful of other restrictions that could be used to circumvent secure boot. 

That was back in early September, and those changes were relatively uncontroversial except for the capability name Garrett chose (`CAP_SECURE_FIRMWARE`) and the `kexec()` restriction. In mid-September, he followed up with a [set of patches](<https://lwn.net/Articles/523420/>) that were substantially the same, though the `kexec()` patch was removed and the capability was renamed to `CAP_COMPROMISE_KERNEL`. In the patch, he noted that ""if anyone wants to deploy these then they should disable kexec until support for signed kexec payloads has been merged"". 

Things went quiet for more than a month, but have since erupted into a rather large thread. A [query from Jiri Kosina](<https://lwn.net/Articles/523424/>) about loading firmware into a secure boot kernel led to a discussion of the threat model that is being covered by Garrett's patch set. While Garrett [agreed](<https://lwn.net/Articles/523426/>) that firmware loading should eventually be dealt with via signatures, it is not as high on his priority list. An automated attack using crafted firmware would be very hardware-specific, requiring reverse engineering, and ""we'd probably benefit from them doing that in the long run"". 

Garrett's focus on automated attacks makes it clear what threat models he is trying to thwart, so Kosina's next query, [about resuming from hibernation](<https://lwn.net/Articles/523431/>), is an issue that Garrett believes should be addressed. It turns out that Josh Boyer has a patch to disable hibernation for secure boot systems, but that, like disabling `kexec()`, was not overwhelmingly popular. 

There are other ways to handle resuming from hibernation, for example by creating keys at boot time that get stored in UEFI boot variables and that the kernel uses to sign hibernation images. But it is clear that some kernel developers are starting (or continuing) to wonder if the kernel secure boot support isn't going a bit--or far more than a bit--overboard. 

For one thing, as James Bottomley [pointed out](<https://lwn.net/Articles/523437/>), there will always be kernel bugs that allow circumvention of these restrictions (e.g. by root reading the hibernation signing key or flipping the capability bit): 

[...] a local privilege elevation attack usually exploits some bug (classically a buffer overflow) which executes arbitrary code in kernel context. In that case, the same attack vector can be used to compromise any in-kernel protection mechanism including turning off the secure boot capability and reading the in-kernel private signing key. 

[...] The point I'm making is that given that the majority of exploits will already be able to execute arbitrary code in-kernel, there's not much point trying to consider features like this as attacker prevention. We should really be focusing on discussing why we'd want to prevent a legitimate local root from writing to the suspend partition in a secure boot environment. 

But kernel exploits appear to be "off the table", at least in terms of the secure boot circumvention that Garrett and others are concerned about. Kosina [said](<https://lwn.net/Articles/523446/>): 

My understanding is that we are not trying to protect against root exploiting the kernel. We are trying to protect against root tampering with the kernel code and data through legitimate use of kernel-provided [facilities] (/dev/mem, ioperm, reprogramming devices to DMA to arbitrary memory locations, resuming from hibernation image that has been tampered with, etc). 

It's not exactly clear why Microsoft would make a distinction between a kernel exploit and using legitimate kernel services when making a blacklisting decision, though. But, for distributions that _do_ ship signed kernels, they can reduce the attack surface substantially: to only those kernels that they have signed, with whatever vulnerabilities are present in those particular versions. 

Eric Paris [detailed one possible attack](<https://lwn.net/Articles/523449/>) that installs a crafted Linux boot environment (with a legitimately signed bootloader and kernel), which sleeps after setting up a trojaned Windows hibernation image. Users would need to wake the machine twice, but would end up running malware in a secure boot system. 

Bottomley and others are, at the very least, uncomfortable with the idea of an "untrusted root". At the heart of the kernel changes for secure boot is removing the ability for root to make persistent changes to the boot environment. The patches that Garrett has proposed close many of the known holes that would allow root to make those kinds of changes, but the argument is that there are likely to be others. As Alan Cox [put it](<https://lwn.net/Articles/523396/>): 

With all the current posted RH patches I can still take over the box as root trivially enough and you seem to have so far abolished suspend to disk, kexec and a pile of other useful stuff. To actually lock it down you'll have to do a ton more of this. 

Another possible way to handle Linux being used as an attack vector against Windows (which is how keys are likely to get blacklisted) is to change the behavior of the Linux bootloaders. Bottomley [suggested](<https://lwn.net/Articles/523455/>) that a "present user" test on the first boot of the bootloader, which could be detected because the UEFI key database and the "machine owner key" database do not contain the proper keys, would alleviate the problem. Garrett [pointed out](<https://lwn.net/Articles/523456/>) that the shim bootloader does not do this because it needs to be able to boot unattended, even on first boot. But, Bottomley [saw that](<https://lwn.net/Articles/523457/>) as unfortunate: 

[...] what I'm telling you is that by deciding to allow automatic first boot, you're causing the windows attack vector problem. You could easily do a present user test only on first boot which would eliminate it. Instead, we get all of this. 

Garrett, though, sees unattended first boot as an absolute requirement, especially for those who are trying to do automated installations for Linux systems. Others disagreed, not surprisingly, and the discussion still continues. It should be noted that the pre-bootloader that Bottomley released _does_ do a present user test on first boot (and beyond, depending on whether the user changes the secure boot configuration). 

There does seem to be something of whack-a-mole problem here in terms of finding all of the ways that this "untrusted root" might be able to impact secure boot. In addition, new kernel features will have to also be scrutinized to see whether they need to be disabled depending on `CAP_COMPROMISE_KERNEL`. 

Not trusting root is a very different model than kernel developers (and users) are accustomed to. One can imagine that all of the different problem areas will be tracked down eventually, but it will be a fair amount of work. Whether that work is truly justified in support of a feature that is largely (though not completely) only useful for protecting Windows is a big question. On the other hand, not being able to [easily] boot Linux on x86 hardware because of key blacklisting would be problematic too. This one will take some time to play out.
