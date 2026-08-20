---
title: "KS2010: Lightning talks"
url: https://lwn.net/Articles/412750/
date: "November 2, 2010"
category: "Big kernel lock; Development tools-Coccinelle; Firmware"
author: "By Jonathan Corbet November 2, 2010 2010 Kernel Summit"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 2, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

The final session on the first day of the 2010 Kernel Summit was dedicated to five-minute talks on specific topics. Some developers seemed to find it one of the more productive sessions of the entire day. 

Julia Lawall started off with a description of [Coccinelle](<https://lwn.net/Articles/315686/>); that, in turn, started with a look at bugs in the kernel code base. Using Coccinelle to find bugs in the kernel showed that, over the last few years, the number of bugs which can be identified by this tool has remained roughly constant. The size of the kernel has grown considerably, though, leading to the conclusion that the quality of the code as a whole is improving. The bug density is going down in most subsystems; the staging tree was notable for having a higher-than-usual number of bugs, though. 

Coccinelle is useful for finding a lot of types of bugs through advanced code analysis and pattern matching. It can finger many types of memory leaks, locking problems, bad API usage, null pointer dereferences, and so on. Arnd Bergmann noted that it can be very useful for refactoring as well; he found it to be helpful in his big kernel lock removal work. 

Andrew Morton suggested that some value could be gained by looking more closely at fixes being merged into the mainline. Many of them could reveal common bug patterns which would be amenable to detection with Coccinelle. Evidently that kind of work is being done now. Rafael Wysocki said that the kernel bugzilla tends to contain detailed information on bugs, their source, and their resolution. Patches to the stable tree were also pointed out as being a good source of information on bugs. 

The talk concluded with a loud round of applause for this work. 

Grant Likely talked about the use of the kernel's device model and its limitations, which are becoming increasingly evident. A lot of code is not using the information on device topology which is there now, and it is proving increasingly difficult to make the device model work with new code. Current hardware tends to have all kinds of complex interrelationships which cannot really be represented in the device model's simple hierarchy. So, Grant asked, is it time to review the device model? Further discussion was deferred to the power management and embedded microconfs to be held at the Linux Plumbers Conference later in the week. 

Arnd Bergmann got up to ask: what should be done about the small number of big kernel lock users remaining in the kernel? Should they be moved to the staging tree? Linus said that he intends to set the default for `CONFIG_BKL` in 2.6.37 to "no." The biggest remaining subsystem with BKL problems, it seems, is Video4Linux; a patch exists and could conceivably even be merged for 2.6.37. The other potential issue is the UDF filesystem; distributors will be unwilling to do without UDF, so the BKL will have to remain until it's fixed. Once that's taken care of, though, the way seems clear for moving any other BKL-using code to the staging tree. If it's not fixed within a few development cycles, that code will then be staged out of the mainline entirely. 

Thomas Gleixner talked briefly about his ongoing rework of the low-level interrupt handling infrastructure. He is just about done with the long-term project of removing `do_IRQ()` from the core. Most of the architectures have already been fixed to not need it. There is a new set of callbacks for interrupt controller chips; Thomas does not intend to preserve the old callbacks past 2.6.38. There is a lot more work to be done for this change, especially in the ARM tree where every CPU variant has its own controller. Much of that work can be done with Coccinelle, though. 

David Woodhouse concluded the session with a brief talk on the firmware tree. Are there any distributions which are still shipping any firmware from the kernel tree, as opposed to the separate firmware tree? Steven Hemminger said that Vyatta still ships in-tree firmware; for a number of network drivers, it tends to be more current than what is found in the firmware tree. 

David would like to remove the remaining in-kernel firmware entirely; among other things, that will force vendors to update firmware in the firmware tree instead. Linus is amenable to that change, as long as the firmware tree's code is at least as current as what's found in the kernel. He said that he's happy to remove code, especially when it's politically sensitive code. So it seems likely that, quite soon, there will be no more firmware blobs in the kernel proper. 

[Next: Linux at NASDAQ](<https://lwn.net/Articles/412818/>) (day 2).
