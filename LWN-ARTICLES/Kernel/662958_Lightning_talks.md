---
title: Lightning talks
url: https://lwn.net/Articles/662958/
date: "November 4, 2015"
category: Android; Year 2038 problem
author: "By Jonathan Corbet November 4, 2015 2015 Kernel Summit"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 4, 2015

* * *

[2015 Kernel Summit](<https://lwn.net/Articles/KernelSummit2015/>)

The lightning talks part of the 2015 Kernel Summit schedule was for topics that were not big enough to justify a full session on their own. 

Josh Triplett started by talking about the [Kernel Tinification project](<https://tiny.wiki.kernel.org/>), which seeks to make it possible to build small kernels for deeply embedded applications. He noted that the zero-day robot, which normally concerns itself with compilation and boot errors, is now able to send out messages for patches that unduly increase the size of the kernel. The news was met with general applause. 

Arnd Bergmann talked a bit about the work toward solving the [year-2038 problem](<https://lwn.net/Articles/599580/>). He has a patch series that simply removes the definition of `time_t` from the kernel; that has the effect of breaking the compilation of every file that uses it. He can then go through the wreckage and look at fixing the problems; he has about 200 patches accumulated so far toward this end. There are, he said, still 113 drivers in need of fixing. He is working on [introducing new system calls](<https://lwn.net/Articles/643234/>) to enable 32-bit systems to survive 2038. 

Most of the fixes are relatively easy, but there are some hard ones as well. Some subsystems will need user-space changes to survive the epoch. One of those is the input subsystem, which uses 32-bit timestamps; there is a plan in place for that one. A number of filesystems need work, and ext4 will need an extended inode format. Some filesystems, he said, will almost certainly never be extended. Some drivers will never work properly, since they write 32-bit timestamps directly into hardware registers. 

Arnd said that, whenever possible, he wants to avoid breaking things. Linus reinforced the importance of that point, saying that, for example, the FAT filesystem should not be disabled even if its times are wrong. In the end, he asked, who cares about FAT timestamps? Similarly, disabling drivers is not acceptable for a production system, though, as a development tool, it can clearly make sense. 

Tim Bird had a quick followup to [the previous day's session](<https://lwn.net/Articles/662147/>) on running mainline kernels on a cellphone. His [slides [PDF]](</images/conf/2015/klf-ks/bird-slides.pdf>) included a table of how much out-of-tree code is found in handset kernels. The numbers vary from 1-3 million lines of code. For the configuration running on those devices, out-of-tree code comprises as much as 2/3 of the entire kernel running there. 

In Sony's case, there is a massive delta between the kernel it ships and mainline. Starting with a 3.4 kernel, Sony adds 1,799 patches of its own. But that pales next to the 20,395 patches from Qualcomm and the 2,677 from "other." Sony would like to participate more in community development, he said, but its kernel is just too far away from the mainline for that to be practical. 

There was a question about how much common code is found in those out-of-tree patches. There is some, evidently, but also lots of duplication of functionality. Thomas Gleixner noted that different trees often have different drivers for the same hardware. 

Tim asserted that the mainline Broadcom wireless driver has never been run on production hardware, but Olof Johansson contradicted him, saying that it is shipped on Chromebooks. Tim added that Sony has backported the driver to the kernels it ships and is working on getting the whole thing working. He also said that nobody has ever charged a mobile device using mainline code; here, Sony is working on a USB-charger framework for the mainline.
