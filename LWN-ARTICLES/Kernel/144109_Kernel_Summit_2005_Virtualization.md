---
title: "Kernel Summit 2005: Virtualization"
url: https://lwn.net/Articles/144109/
date: "July 19, 2005"
category: Virtualization; Xen
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
Rik van Riel and Chris Wright discussed virtualization. This is currently a hot topic, and several OLS sessions will be dedicated to it in the next few days. At the kernel level, however, it seems that there are not a whole lot of issues which need to be resolved. 

Virtual hosts may have multiple virtual processors; they will schedule processes between them. The physical host may also have multiple processors, and it will be performing its own scheduling. Since the two levels know little about each other, scheduling imbalances can result. Xen does a certain amount of "rotating" processes around to deal with this problem. Despite being discussed for a while, this issue does not appear to be particularly serious. 

It was pointed out that the various virtualization implementations (Xen and user-mode Linux in particular) have their own virtual buses, virtual drivers, etc. Might there be some benefit in merging them? Perhaps, but the amount of code involved is quite small. 

Merging Xen. The Xen patches create a completely new architecture for the virtual machine. There have been objections to this approach; it looked like a maintenance problem, especially as Xen is ported to more real architectures. So the patches are being reworked, and the `arch/xen` directory is going away. Stuff which is truly specific to Xen will find its way into the `drivers` or host architecture directories. With these changes made, opposition to the merging of Xen should be much reduced. 

Linus had to ask: is anybody actually using Xen? The biggest users are, as expected, in the virtual hosting business. Most of them are still in relatively early evaluation stages - Xen is a young technology. Xen is also heavily used by people who want to play with multiple distributions or otherwise need sandbox systems to work with.
