---
title: USB and fast booting
url: https://lwn.net/Articles/330378/
date: "April 29, 2009"
category: Bootstrap process
author: "By Jake Edge April 29, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 29, 2009

The changes that are being made for a faster-booting Linux have generally been welcomed, but when they lead to an apparent regression, complaints will be heard. That situation arose recently when Jeff Garzik [reported a regression](<https://lwn.net/Articles/330779/>) that caused one of his systems to no longer boot. Because of some changes made to the way USB initializes, the system no longer recognized his disks in time to mount the root filesystem from them. As it turns out, the problem is not limited to disks, nor is it new; it is a longstanding race condition that previously was being "won" by most hardware, but that same hardware is often losing the race now. 

Garzik had bisected the problem to a particular [commit](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commit;h=8520f38099ccfdac2147a0852f84ee7a8ee5e197>) made back in September of 2008. Instead of sleeping for 100ms as part of the initialization of each USB root hub, the new code uses the delayed work mechanism to schedule the next initialization step 100ms in the future. For kernels which had the USB code built-in, this would allow the boot thread to do other work, rather than block waiting for these delays. It had a rather positive impact on boot speed, with patch author Alan Stern reporting: 

Arjan van de Ven was very pleased to see that this shaved 700 ms off his computer's boot time. Since his total boot time is on the order of two seconds, the improvement is considerable. 

From Garzik's perspective, the problem is that this system booted successfully with every kernel version until 2.6.28. The immediate suggestion was to use the `rootdelay` kernel boot option which will delay the boot process for the given number of seconds before trying to mount the root filesystem. That did not sit very well with Garzik, and he [asked](<https://lwn.net/Articles/330807/>): ""When did regressions become an acceptable tradeoff for speed?"" 

As it turns out, Garzik had just been "lucky" before, he could have run into this problem on earlier kernels with different hardware as Greg Kroah-Hartman [points out](<https://lwn.net/Articles/330810/>): ""What happens when you buy a new box with more USB host controllers and a faster processor? Same problem."" The underlying issue is specific to USB, as the old initialization waited 100ms per USB bus (i.e. root hub) synchronously, so a system with five hubs would effectively wait 500ms for the first to initialize and enumerate the devices attached. The new code does those same initializations in parallel. 

While it is relatively uncommon to have USB root filesystems, it is far from unheard of. Embedded systems are a fairly likely candidate, due to cost and form factor issues, as Alan Cox [explained](<https://lwn.net/Articles/330894/>). Multiple distributions also have support for running completely from a USB device, typically a USB flash drive. 

But, as Garzik and others point out, users that upgrade their kernels (or distributions who do so), but don't add in a `rootdelay` option, risk having systems that cannot boot. USB is fundamentally different than other buses, however, because there is no way to know when the enumeration of devices on a particular hub has been completed. Mark Lord [questioned](<https://lwn.net/Articles/330837/>) the explanation, noting: ""SATA drives also take variable amounts of time to 'show up' at boot."" But as Arjan van de Ven [explained](<https://lwn.net/Articles/330839/>), there is a significant difference: 

the difference is that with sata you know when you are done and have all possible drives. No so much much with USB. So with SATA we can, and do, wait for the scan to complete at the right point in the boot. 

It turns out that the same problem in a slightly different guise shows up for embedded devices that use USB consoles. David VomLehn has been working on a [patch](<https://lwn.net/Articles/329179/>) to wait for USB consoles to become available. Because embedded devices often have USB consoles, but only for development and debugging, a long delay waiting for a console that is unlikely to show up in the majority of cases is undesirable. But, because it is impossible to know that all USB devices have reported in, some kind of delay is inevitable. VomLehn's mechanism would delay up until a timeout specified in the kernel boot parameters, but, unlike `rootdelay`, would wake up early as soon as a console device was detected. 

As VomLehn [notes](<https://lwn.net/Articles/330848/>), the problem goes even further than that, affecting USB network devices needed at boot time as well. Discussion on various versions of his patch also pointed out that similar problems exist for other buses. As boot parallelization gets better--and more pervasive--more of these kinds of problems are going to be discovered. A more general solution for devices required at boot time needs to be found as van de Ven [describes](<https://lwn.net/Articles/330850/>): 

For other pieces it's hard. Non-enumeratable busses just suck; at some point all you can do is just wait (which we have already available today for anyone to do). I realize people don't want to just wait 4 seconds (the people who first objected to boot time improvements then suddenly care about boot time ;-)... 

For root fs there's some options, and I have patches to basically retry on fail. (The patches have a bug and I don't have time to solve it this week, so I'm not submitting them) For other devices it is hard. Realistically we need hotplug to work well enough so that when a device shows up, we can just hook it up when it does. 

So far, the problems have just been identified and discussed. Workarounds like `rootdelay` have been mentioned, but that only "solves" one facet of the problem. Distributions are, or will be, shipping 2.6.29 kernels in their upcoming releases, one hopes they have already dealt with the issue or there may be a number of rather puzzled users with systems that don't boot. It would seem important to address the problems, at least for USB storage, as part of 2.6.31.
