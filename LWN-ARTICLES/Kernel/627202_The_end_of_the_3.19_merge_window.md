---
title: The end of the 3.19 merge window
url: https://lwn.net/Articles/627202/
date: "December 22, 2014"
category: "Releases-3.19"
author: "By Jonathan Corbet December 22, 2014"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 22, 2014

By the usual practice, the 3.19 merge window should have ended on December 21, two weeks after its opening. But the solstice celebration featuring 3.19-rc1 was not to be; Linus decided to [close the merge window](<https://lwn.net/Articles/627181/>) one day early. Among other things, his reasoning was that so much work had come in that there wasn't a whole lot of point in waiting for more: 

Considering how much came in fairly late, I find it hard to care about anybody who had decided to cut it even closer than some people already did. That said, maybe there aren't any real stragglers - and judging by the size of rc1, there really can't have been much. 

Indeed, this is a busy development cycle, with 11,408 changesets having been pulled during the merge window. That is more than was seen during the entire 3.18 development cycle, where 11,379 changesets were merged before the final release. 

About 1000 changesets were pulled since [last week's summary](<https://lwn.net/Articles/626150/>); some of the more interesting, user-visible changes in that set were: 

  * The handling of the `setgroups()` system call in user namespaces has been changed in a way that could possibly break some applications; see [this article](<https://lwn.net/Articles/626665/>) for more information. 

  * The Ceph filesystem now supports inline data, improving performance for small files. Ceph also supports message signing for authentication between clients and servers. 

  * KVM virtualization support for the Itanium (ia64) architecture has been removed. It was not being maintained, and, seemingly, was not being used either. 

  * The InfiniBand layer has gained support for on-demand paging. This feature allows an [RDMA](<https://en.wikipedia.org/wiki/Remote_direct_memory_access>) region to be set up and populated via page faults when the memory is actually used, thus avoiding the need to pin down a bunch of memory that may never be needed. 

  * New hardware support includes: 

    * **Input** : Elan I2C/SMbus trackpads, Goodix I2C touchscreens, and Elan eKTH I2C touchscreens. 

    * **Miscellaneous** : Allwinner SoC-based NAND flash, Broadcom BCM2835 pulse-width modulator (PWM) controllers, and Atmel HLCDC PWM controllers. 

    * **Thermal** : Support for generic device cooling through clock frequency tweaking, NVIDIA Tegra SOCTHERM thermal management systems, and Rockchip thermal management systems.  

Changes visible to kernel developers include: 

  * The module removal code has been reworked to get rid of calls to the much-maligned `stop_machine()` function. In the process, module reference counting has been slowed slightly, but it is believed that nobody is tweaking reference counts often enough that the difference will be noticed. 

  * The `CONFIG_PM_RUNTIME` configuration symbol has finally been eliminated from the kernel; everything uses `CONFIG_PM` now. 

  * The `READ_ONCE()` and `ASSIGN_ONCE()` macros (described in [this article](<https://lwn.net/Articles/624126/>)) were merged in the final pull before the closing of the merge window. These macros enforce the use of ~~non-~~ scalar types, hopefully avoiding experiences with tricky compiler bugs. 

If the usual schedule holds, the final 3.19 release can be expected sometime around the middle of March. Between now and then, though, a lot of testing and fixing has to be done — 11,400 new changes will certainly have brought a few bugs with them. That said, the early indications are that 3.19-rc1 is relatively stable for such an early release, so this may yet turn out to be another fast cycle despite the volume of patches.
