---
title: NDISwrapper dodges another bullet
url: https://lwn.net/Articles/271762/
date: "March 5, 2008"
category: Copyright issues; NDISwrapper
author: "By Jake Edge March 5, 2008"
---

By **Jake Edge**  
March 5, 2008

Hardware compatibility has long been a problem for Linux--though it has gotten much better over the years--so it will be surprising to some to see a kernel change that will make some hardware cease working. For others, who follow kernel development a bit more closely, it will come as no great surprise that [NDISwrapper](<http://ndiswrapper.sourceforge.net/>) was disabled by a change made to the kernel back in January. NDISwrapper has never been very popular with kernel hackers, but, because it is GPL licensed and allows more hardware to be used, there are folks on both sides of the argument. For a while, it looked like NDISwrapper had lost that argument, but the 2.6.25-rc4 release restores the functionality it requires. 

NDISwrapper is a kernel module that is used to load Windows-only drivers into Linux. For some hardware, notably wireless network cards, it is the only way to support them because the manufacturer provides neither specifications nor a working Linux driver. Unfortunately, many of these cards are installed in laptops where it is difficult or impossible to replace them with Linux-friendly alternatives. This is what led to implementing the Network Device Interface Specification (NDIS) for Linux. NDIS is an ancient--it was originally developed by Microsoft and 3Com for MS-DOS in the mid to late 1980s--interface for networking devices, which is still in use today. 

The NDISwrapper code has been around since 2003, but always as a separate module that must be built by the user (or distribution) and loaded into the kernel. It is not part of the mainline kernel, nor will it ever be; maintaining a glue layer that allows proprietary, closed-source drivers to be linked into the kernel is not high on anyone's list. But, NDISwrapper _is_ GPL. Its code is available for inspection or modification by all, so that is not the problem, it is the intent that matters. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

When a binary-only driver--the NVidia video driver for example--is loaded into the kernel, a "taint" flag is set, indicating that the kernel is tainted by code that cannot be examined. Bug reports for tainted kernels are routinely ignored, unless they can be reproduced in an untainted kernel. Life, it seems, is too short to try and diagnose problems that could easily have been created by a buggy driver that cannot be debugged. Originally, the taint flag was just a means to detect and ignore those bug reports, but over time it has become part of a mechanism to restrict which symbols a module can access. 

Some kernel symbols are considered so integral that any module using them must be a derivative work. Therefore, modules that want to use them must be GPL. Modules declare their license using the `MODULE_LICENSE` macro, while symbols are exported using either `EXPORT_SYMBOL` or `EXPORT_SYMBOL_GPL`. Any module that doesn't have a compatible license doesn't get access to the GPL-only symbols. 

Few would argue for a GPL module which existed to re-export all of the GPL-only symbols to non-GPL modules. But that is not what NDISwrapper does; instead it implements NDIS, but in order to do that, needs access to GPL-only symbols, mostly for USB and workqueue interfaces. It would be hard to contend that NDIS drivers are derivative of the Linux kernel, they were written for an entirely different system using an interface that predates Linux. This is why NDISwrapper developers and users think that an exception should be made for it. Clearly the Windows drivers taint the kernel, but accessing a subset of the GPL-only functionality through NDISwrapper should be allowed, they argue. 

Since NDISwrapper itself is GPL, the normal module loading rules would allow it to access GPL-only symbols, except that an explicit check for NDISwrapper was added to the 2.6.16 kernel. The question, then, revolves around what should be done when the kernel detects it being loaded. NDISwrapper has always been careful to mark the drivers that it loads as tainted, but the recent patch marks the module itself as tainted, disallowing access to the GPL-only symbols and breaking NDISwrapper. Absent that patch, only the kernel is marked as tainted--the module itself is not. 

A similar situation occurred back in October 2006, which LWN [covered on the Kernel page](<http://lwn.net/Articles/205644/>), when a stricter interpretation of tainting started to be enforced. At that point, NDISwrapper stopped working and it looked like it might stay that way, until Andrew Morton stepped in with [objections to breaking NDISwrapper with no warning](<http://article.gmane.org/gmane.linux.kernel/460618>). Shortly thereafter, a patch was merged that only marked the kernel as tainted when NDISwrapper is loaded. At that point, the issue fell by the wayside, until now. 

Part of the problem is that marking a symbol as GPL-only means different things to different developers. For some, it is a means to warn proprietary driver developers that they are straying into territory that makes distribution of their drivers very likely to be a violation of the GPL, while others want to use it to completely eliminate binary-only kernel drivers. There is no policy that clearly delineates which interpretation is "correct". Meanwhile, NDISwrapper has been in use by many for four years or more; breaking it now, with little or no warning, is likely to create some very unhappy users. 

Linus Torvalds clearly thinks there are [no licensing issues with NDISwrapper](<https://lwn.net/Articles/271789/>): 

Quite frankly, my position on this has always been that the GPLv2 explicitly covers _derived_ works only, and that very obviously a Windows driver isn't a derived work of the kernel. So as far as I'm concerned, ndiswrapper may be distasteful from a technical and support angle, but not against the license. 

Jon Masters, the author of the patch that inadvertently made this change, had an [excellent suggestion](<https://lwn.net/Articles/271791/>) that should be pursued to try and reduce these kinds of problems in the future: 

Since we've brought it up, one good thing I would like to see come of this perhaps is a clearer understanding of what the kernel should and should not be doing in terms of "license compliance enforcement". We have had lots of talk, but perhaps a "policy" document is worthwhile. 

Another interesting battle will be that surrounding [exporting `init_mm()`](<http://article.gmane.org/gmane.linux.kernel/646638>) which was removed in early versions of 2.6.25, but then restored in 2.6.25-rc4. It is fairly clearly a low-level kernel interface that is unused by any in-tree driver, so its export was removed. One rather glaring exception is that the out-of-tree NVidia binary drivers _do_ use it. Its export has been restored for one more development cycle, but it is clearly seen as something that should not be touched by drivers. It could be quite a struggle between the developers and users of a very popular driver and the kernel hackers that don't want to see kernel API abuse. 

Issues surrounding the GPL are always contentious on linux-kernel; this one is no different. While NDISwrapper is an out-of-tree driver, it has hardly been invisible, so complaints when it breaks should come as no surprise. A simple renaming will avoid the current kernel check, so breaking it that way will mostly be an annoyance to users rather than a real barrier to its use. Since there is no real consensus amongst kernel hackers on the binary driver issue, it is hard to see one emerging with regards to NDISwrapper, but that would be the best outcome. One way or another, it needs to be decided, NDISwrapper shouldn't come under a periodic threat of breaking. If it is determined to be a violation of the kernel interfaces, that should be clearly indicated and its users should be given some warning so they can find alternatives.
