---
title: The proper use of EXPORT_SYMBOL_GPL()
url: https://lwn.net/Articles/769471/
date: "October 27, 2018"
category: "Copyright issues; EXPORT SYMBOL GPL; Memory management-Heterogeneous memory management"
author: "By Jonathan Corbet October 27, 2018 Maintainers Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 27, 2018

* * *

[Maintainers Summit](<https://lwn.net/Articles/769260/>)

The kernel, in theory, puts strict limits on which functions and data structures are available to loadable kernel modules; only those that have been explicitly exported with `EXPORT_SYMBOL()` or `EXPORT_SYMBOL_GPL()` are accessible. In the case of `EXPORT_SYMBOL_GPL()`, only modules that declare a GPL-compatible license will be able to see the symbol. There have been questions about when `EXPORT_SYMBOL_GPL()` should be used for almost as long as it has existed. The latest attempt to answer those questions was a session run by Greg Kroah-Hartman at the 2018 Kernel Maintainers Summit; that session offered little in the way of general guidance, but it did address one specific case. 

The kernel has had `EXPORT_SYMBOL_GPL()` for fifteen years now, Kroah-Hartman said; its use is not mandatory. It is generally meant to [![\[Greg
Kroah-Hartman\]](https://static.lwn.net/images/conf/2018/ms/GregKroahHartman-sm.jpg)](<https://lwn.net/Articles/769472/>) apply to core functions that cannot be used without the user being a derived work of the kernel. But whether that is the case for specific functions is not always obvious. 

Andrew Morton was quick to raise the case that has been concerning him, relating to [symbols exported for the heterogeneous memory management (HMM) subsystem](<https://lwn.net/Articles/757124/>). In particular, it makes some low-level memory-management functionality available to all modules, rather than just those with a GPL-compatible license. This export, Morton said, is "a big gift to NVIDIA", which needs it to use the HMM functionality in its closed-source modules. This export has upset a number of people including Dan Williams, who has been posting patches to change that export to `EXPORT_SYMBOL_GPL()`. 

Morton said that he didn't really want to get into the politics of the situation, but he needed to decide whether to apply Williams's patches, and that means deciding whether a GPL-only export would be more appropriate in this case. Christoph Hellwig was quick to argue that any users of the functionality in question can only be a derived work of the kernel. Linus Torvalds said that the initial point was to let hardware with its own memory-management unit handle its own page-table management, but that is not how the usage has actually turned out. 

Hellwig said that there is other NVIDIA-specific code in the kernel that should probably be removed as well; support for [NVLink](<https://en.wikipedia.org/wiki/NVLink>) was mentioned in particular. Arnd Bergmann said that there is a smaller pile of patents around AI applications (where NVLink is generally used) than around graphics, so there might be a better chance of getting that code opened eventually. Graphics drivers remain a problem, though. 

Returning to the HMM issue, Morton summarized the feeling in the room as being in favor of merging Williams's patches. So, he said as the session (and the summit as a whole) came to a close, that is what he will do. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to the Maintainers Summit.]
