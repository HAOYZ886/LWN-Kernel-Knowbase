---
title: "Reverting disk aliases?"
url: https://lwn.net/Articles/467480/
date: "November 16, 2011"
category: Device naming
author: "By Jake Edge November 16, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
November 16, 2011

Back in June, we [looked](<https://lwn.net/Articles/448505/>) at a proposed mechanism for adding aliases to device names, disks in particular. Since then, the patch has been merged into the mainline, but some kernel developers are not happy with that and have asked that it be reverted. Part of the complaint is that the functionality adds to the kernel ABI, which will need to be maintained "forever", but there are other solutions to the problem that don't require kernel changes. So far, the patch has not been reverted, but there is an underlying question: who gets to decide when and where to extend the kernel's ABI? 

The [alias patch](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commit;h=a72c5e5eb738033938ab30d6a634b74d1d060f10>) was authored by Nao Nishijima and came into the mainline (for 3.2-rc1) by way of James Bottomley's SCSI tree. The patch allows administrators to associate an alias name for a particular disk by writing to the `/sys/block/<disk>/alias` sysfs file. That way, certain log messages can be made using the user-supplied disk name rather than the raw name of the disk, which may change on each boot. 

Tejun Heo [requested](<https://lwn.net/Articles/467573/>) that the patch be reverted, noting that it ""has been nacked by people working on device driver core, block layer and kernel-userland interface and shouldn't have been upstreamed"". That request was quickly acked by several people (Greg Kroah-Hartman, Kay Sievers, Jens Axboe, and Jeff Garzik), with Axboe explicitly noting that it should be done soon: ""We need to revert it before 3.2 rolls out, otherwise we are stuck with it."" 

As might be guessed, though, Bottomley [disagreed](<https://lwn.net/Articles/467584/>) that it should be reverted, saying that it solved a real problem: 

No, I can't agree with this ... if you propose a working alternative, I'm listening, but in the absence of one, I think the hack fills a gap we have in log analysis and tides us over until we have an agreed on proper solution (at which point, I'm perfectly happy to pull the hack back out). 

Several folks pounced on the "hack" admission in Bottomley's note, but both Kroah-Hartman and Sievers believe that there is no need for a kernel-side solution at all. As Sievers [put it](<https://lwn.net/Articles/467586/>): 

The solution to this problem is to let udev log the known symlinks to the log stream at device discovery time. That way you can log _all_ kernel device messages to the currently [known] disk names. This works already even on old systems, 

Furthermore, Kroah-Hartman [pointed out](<https://lwn.net/Articles/467590/>) that Nishijima recognizes that it can be solved in user space: ""Again, this is fixable in userspace, the author of the patch agrees with that, yet refuses to make the userspace changes despite having a few _years_ in which to so so"". As with the others commenting, Sievers is also concerned about adding to the user-space interface: ""Such hacks are not supposed to get in, and its really hard to get them out again."" 

While the patch has not been reverted, Nishijima may be anticipating that outcome with a [post](<https://lwn.net/Articles/467603/>) that looks at changes to udev: ""I understood why this patch is not acceptable and would like to solve the problem of the device name mismatch in *user space* using udev"". Kroah-Hartman suggests posting udev patches that implement the changes to the linux-hotplug mailing list as a good starting point. 

It would seem that Bottomley made something of an end-run around the objections of various maintainers by pulling the change into his tree. His reasons for doing so make sense, because there are customers asking for the change, but it still routes around the usual paths. Heo's request certainly indicates that he doesn't believe it came in via the proper path, and Kroah-Hartman is blunt about that as well: ""Also, you should have gotten this through the block layer maintainer..."". It _is_ a hack as everyone seems to agree, but it's a hack that leaves behind an ABI for the kernel to maintain forevermore. It is not surprising that a number of core developers would like to see it reverted.
