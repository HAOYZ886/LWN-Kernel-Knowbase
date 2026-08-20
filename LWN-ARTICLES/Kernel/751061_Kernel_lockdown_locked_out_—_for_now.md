---
title: Kernel lockdown locked out — for now
url: https://lwn.net/Articles/751061/
date: "April 6, 2018"
category: "Security-UEFI secure boot"
author: "By Jonathan Corbet April 6, 2018"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 6, 2018

As the 4.17 merge window opened, it [seemed possible](<https://lwn.net/Articles/750730/>) that the kernel lockdown patch set could be merged at last. That was before the linux-kernel mailing list got its hands on the issue. What resulted was not one of the kernel community's finest moments. But it did result in a couple of evident conclusions: kernel lockdown will almost certainly not be merged for 4.17, but something that looks very much like it is highly likely to be accepted in a subsequent merge window. 

As a reminder: the purpose of the lockdown patches is to enforce a distinction between running as root and the ability to run code in kernel mode. Proponents of UEFI secure boot maintain that this separation is necessary; otherwise the promise of secure boot (that the system will only run trusted code in kernel mode) cannot be kept. Closing off the paths by which a privileged attacker could run arbitrary code in kernel mode requires disabling a number of features in the kernel; see the above-linked article for the details. Most users will never miss the disabled features, but there are always exceptions. 

There are, naturally, a number of disagreements on how the lockdown mode is implemented. The use of a blacklist to disable "dangerous" kernel command-line options seems sure to let some of those options through. It is unlikely that all of the potentially hazardous operations supported by device drivers can ever be found. And so on. 

The interesting thing, though, is that almost nobody seems to object to the lockdown concept in general — as long as it can be turned off. Even Linus Torvalds, who argued against the lockdown patches and their developers in typical Torvalds style, [sees some potential value](<https://lwn.net/Articles/751066/>) in the lockdown concept. There does not appear to be any significant opposition to making it available in the kernel. 

#### The sticking point

The reason that the lockdown patches will not be merged this time around thus doesn't depend on their core purpose. Instead, the whole thing hinges on a single detail: the patch set automatically turns the lockdown mode on if secure boot is detected at startup time. It is the tying together of lockdown and secure boot that brought about a long and unpleasant linux-kernel thread. 

Torvalds [pointed out](<https://lwn.net/Articles/751067/>) that there is a long list of security-related features that can be enabled in current kernels. None of those features depend on whether secure boot is enabled on the system; they are configured in or out on their own merits. The behavior of the kernel should not vary as a result of a BIOS setting, he argued. He also [claimed](<https://lwn.net/Articles/751068/>) that connecting the two features means that few kernel developers will ever test kernels with lockdown enabled, since few of them enable secure boot on their development systems. No ""sane distribution"" would ship a kernel with this mode turned on, he [said](<https://lwn.net/Articles/751069/>). One little problem with that last claim, as Matthew Garrett [pointed out](<https://lwn.net/Articles/751116/>), is that many major distributions have been shipping a version of this patch set for about five years. 

On the other side, proponents argue that lockdown without secure boot (or something like it) will instill a false sense of security, since the lockdown can be circumvented by attacking the boot chain. As Garrett [put it](<https://lwn.net/Articles/751115/>): ""Without some sort of verified boot mechanism, lockdown is just security theater"". The same is said to be true of a kernel that supports secure boot without lockdown; that kernel can be compromised after boot to run untrusted code in kernel mode — exactly the scenario that secure boot is meant to prevent. A secure-boot kernel without lockdown is not only false security for its user; it is [presented](<https://lwn.net/Articles/751117/>) as a threat to others as well: 

Because a kernel signed with a generally trusted key that doesn't implement any lockdown functionality is effectively a bootloader that will load unsigned material on most machines on the market, which reduces the security of users running those machines with Secure Boot enabled. 

See also [this blog post](<https://mjg59.dreamwidth.org/50577.html>) from Garrett describing his view of this discussion in more detail. 

Not everybody agrees that lockdown without secure boot is useless; they see it like all of the other hardening technologies that have been put into the kernel. Compromising the boot chain (and forcing a reboot) is not always an easy thing to do, especially for a remote attacker. Secure boot is unlikely to ever protect all of the places where a persistent exploit could be placed anyway — init scripts, for example. Even without secure boot, it is argued, lockdown raises the bar for a potential attacker. 

The "bootloader" argument is an interesting one; it says that a kernel without lockdown can be compromised and used to load a new, modified kernel that hides any malware it contains. [According to Peter Jones](<https://lwn.net/Articles/751118/>), this is a common model for malware installations. Under this line of reasoning, any kernel that can be corrupted in this way and carries a signature that will enable it to boot on a secure-boot system can be used to attack any system that trusts the signing key. Automatically enabling lockdown with secure boot is a way to avoid creating this kind of attack tool. 

If that is the issue, [Alan Cox said](<https://lwn.net/Articles/751120/>), then we have already lost: 

Vendors of all OS's have released enough buggy but signed kernel images over the past years that rummaging around in the archive will find you a wide choice of signed boot images that'll then let you do wtf you like including chaining some other target. 

One other aspect of this issue that came up briefly is the fear that, if Linux looks like a tool that can be used to compromise secure-boot systems running Windows, that Microsoft might blacklist the signing key and render Linux unbootable on most x86 hardware. David Howells [expressed this worry](<https://lwn.net/Articles/751122/>), for example. Greg Kroah-Hartman [said](<https://lwn.net/Articles/751123/>), though, that he has researched this claim numerous times and it has turned out to be an ""urban myth"". 

#### Resolution?

Toward the end of the discussion, Torvalds (and others) suggested that lockdown should just be enabled unconditionally, especially since distributors have been shipping it for some time. The problem with that, of course, is that lockdown does occasionally break a working system. In such cases, users have been advised by distributors to disable secure boot as the easiest solution. Torvalds [rephrased that point](<https://lwn.net/Articles/751145/>): 

We'd like to just enable it all the time, but it's known to break some unusual hardware cases that we can't fix in software, and we wanted *some* way to disable it that requires explicit and verified user intervention to do that, and disabling secure boot is the easiest hack we could come up with. 

Had things been expressed that way from the beginning, he said, the connection between lockdown and secure boot would have been ""much more palatable"". 

A statement like that strongly suggests that the lockdown feature, even perhaps with a secure-boot connection, should be able to get past Torvalds eventually. Howells has [said](<https://lwn.net/Articles/751148/>) that he is reworking the patch set to loosen that connection, which may help as well. While another attempt to push this work for 4.17 could happen, it seems more likely that everybody will want to step away from this discussion and address the issue again in 4.18.
