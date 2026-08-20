---
title: "PCIe, power management, and problematic BIOSes"
url: https://lwn.net/Articles/449448/
date: "June 29, 2011"
category: Power management
author: "By Jonathan Corbet June 29, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 29, 2011

Back in April, Phoronix [announced with some fanfare](<http://www.phoronix.com/scan.php?page=article&item=linux_mobile_uffda&num=1>) that the 2.6.38 kernel - and those following - had a "major" power management regression which significantly reduced battery life on some systems. This problem has generated a fair amount of discussion, including [this Launchpad thread](<https://bugs.launchpad.net/ubuntu/+source/linux/+bug/760131>), but little in the way of solutions. Phoronix now [claims to have located the change that caused the problem](<http://www.phoronix.com/scan.php?page=article&item=linux_2638_aspm&num=1>) and has provided a workaround which will make things better for some users. But a true fix may be a while in coming. 

As a result of the high clock rates used, PCI-Express devices can take a lot of power even when they are idle. "Active state power management" (ASPM) was developed as a means for putting those peripherals into a lower power state when it seems that there may be little need for them. ASPM can save power, but the usual tradeoff applies: a device which is in a reduced power state will not be immediately available for use. So, on systems where ASPM is in use, access to devices can sometimes take noticeably longer if those devices have been powered down. In some situations (usually those involving batteries) this tradeoff may be acceptable; in others it is not. So, like most power management mechanisms, ASPM can be turned on or off. 

It is a bit more complicated than that, though; on x86 systems, the BIOS also gets into the act. The BIOS is charged with the initial configuration of the system and with telling the kernel about the functionality that is present. One bit of information passed to the kernel by the BIOS is whether ASPM is supported on the system. The kernel developers, reasonably, concluded that, if the BIOS says that ASPM is not supported, they should not mess with the associated registers. It turns out, though, that this approach didn't quite work; thus, in December, Matthew Garrett committed [a patch](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=2f671e2dbff6eb5ef4e2600adbec550c13b8fe72>) described as: 

We currently refuse to touch the ASPM registers if the BIOS tells us that ASPM isn't supported. This can cause problems if the BIOS has (for any reason) enabled ASPM on some devices anyway. Change the code such that we explicitly clear ASPM if the FADT indicates that ASPM isn't supported, and make sure we tidy up appropriately on device removal in order to deal with the hotplug case. 

In other words, sometimes the BIOS will tell the system that ASPM is not supported even though ASPM support is present; for added fun, the BIOS may enable ASPM on some devices (even though it says ASPM is not supported) before passing control to the kernel. There are reasons why operating system developers tend to hold BIOS developers in low esteem. 

Had Andrew Morton read the above changelog, he certainly would have complained that "can cause problems" is a rather vague justification for a change to the kernel. Your editor asked Matthew about the problem and got [an informative response](<https://lwn.net/Articles/449648/>) that is worth reading in its entirety: 

If this bit is set, the platform is indicating to the OS that it doesn't support ASPM. In the past we took that to mean that we simply shouldn't touch the ASPM bits. However, it turns out that there's some systems where the BIOS has enabled ASPM itself, set the "ASPM unsupported" bit and then the hardware falls over when an ASPM transition occurs. The most straightforward thing to assume was that the BIOS was stupid (which is, to be fair, my default assumption) and shouldn't have enabled ASPM. So, since that patch, we clear the ASPM state when the BIOS indicates that the platform doesn't support ASPM. 

It's not hard to imagine that putting devices into a state that the kernel was told should not exist might create confusion somewhere. Some research turns up, for example, [this bug report](<https://bugzilla.redhat.com/show_bug.cgi?id=681017>) about system hangs which are fixed by Matthew's patch. If the BIOS says that ASPM is not supported, it would seem that ensuring that no devices think otherwise would make sense. 

That said, this patch is the one that the bisection effort at Phoronix has fingered as the cause of the power regression. Apparently, the notion that disabling low-power states in hardware may lead to increased power consumption also makes sense. The workaround suggested in the article is to boot with the `pcie_aspm=force` option; that forces the system to turn on ASPM regardless of whether the BIOS claims to support it. This workaround will undoubtedly yield better battery life on some affected systems; others may well not work at all. In the latter case, the system may simply lock up - a state with even worse latency characteristics combined with surprisingly bad power use. So this workaround may be welcomed by users who have seen their battery life decline significantly, but it is not a proper solution to the problem. 

Finding that proper solution - preferably one which Just Works without any need for special boot parameters - could be tricky. Quoting Matthew again: 

What alternatives are there? We could keep the status quo and add driver whitelisting for hardware setups that are known to work. The problem is that even where we have specifications for the hardware, we often don't have the errata lists. We don't know for sure whether it works or not. We could revert this patch and add more driver blacklisting. But then we need to track down every device that doesn't work. Or, it's possible that the original code was correct and Linux simply programs the hardware differently, triggering ASPM issues that aren't seen elsewhere. 

Given the uncertainty in the situation, the kernel developers have reached the conclusion that "waste a bit of power" is a lesser evil than "lock up on some systems." In the absence of a better understanding of the problem, any other approach would be hard to justify. So some users may have to use the `pcie_aspm=force` workaround for a while yet. 

Meanwhile, the power usage problem has, as far as your editor can tell, never been raised on any kernel development mailing list. It never appeared in the 2.6.38 regression list. So this issue was invisible to much of the development community; it's not entirely surprising that it has not received much in the way of attention from developers. For better or for worse, the development community has its way of dealing with issues. Reporting a bug to linux-kernel certainly does not guarantee that it will get fixed, but it does improve the odds considerably. Had this issue been brought directly to the developers involved, we might have learned about the root cause some time ago.
