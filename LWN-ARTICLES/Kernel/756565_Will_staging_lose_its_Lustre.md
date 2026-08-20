---
title: "Will staging lose its Lustre?"
url: https://lwn.net/Articles/756565/
date: "June 6, 2018"
category: "Filesystems-Lustre; Staging tree"
author: "By Jonathan Corbet June 6, 2018"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 6, 2018

The kernel's staging tree is meant to be a path by which substandard code can attract increased developer attention, be improved, and eventually find its way into the mainline kernel. Not every module graduates from staging; some are simply removed after it becomes clear that nobody cares about them. It is rare, though, for a project that is actively developed and widely used to be removed from the staging tree, but that may be about to happen with the [Lustre](<http://lustre.org/>) filesystem. 

The staging tree was [created](<https://lwn.net/Articles/285594/>) almost exactly ten years ago as a response to the ongoing problem of out-of-tree drivers that had many users but which lacked the code quality to get into the kernel. By giving such code a toehold, it was hoped, the staging tree would help it to mature more quickly; in the process, it would also provide a relatively safe place for aspiring kernel developers to get their hands dirty fixing up the code. By some measures, staging has been a great success: it has seen nearly 50,000 commits contributed by a large community of developers, and a number of drivers have, indeed, shaped up and moved into the mainline. The "ccree" TrustZone CryptoCell driver [graduated from staging](<https://lwn.net/Articles/749676/>) in 4.17, for example, and the visorbus driver moved to the mainline in 4.16. 

Other code has been less fortunate, though. The gdm72xx, dgap, and olpc_dcon drivers were all deleted in 4.6 due to a lack of interest, and a whole set of RDMA drivers was deleted in 4.5. The COMEDI driver set has received over 8,500 changes since it entered the staging tree, but has still not managed to graduate; it has seen less than 100 patches in the last year. Placement in the staging tree is clearly not a guarantee that a driver will improve enough to move into the mainline. 

Then there is the Lustre filesystem, which was added to the staging tree just over five years ago for the 3.11 release. Lustre has a rather longer history than that, though; it was started by the prolific Peter Braam in 1999\. It was eventually picked up by Sun Microsystems, then suffered death by Oracle in 2010. In more recent times, its development has been managed by [OpenSFS](<http://opensfs.org/>); it seems to have a strong following in industries needing a high-end distributed filesystem for high-performance computing applications. 

As of 4.17, there have been 3,778 patches applied to the Lustre filesystem in the staging tree. A full 33% of those have come from Intel employees, and 11% from Outreachy interns. But this work has not yet managed to make Lustre ready to move out of the staging tree, and the associated [`TODO` file](<https://lwn.net/Articles/756661/>) remains long. It's not clear when Lustre will be brought into shape. 

Indeed, it may never happen. Greg Kroah-Hartman, the maintainer of the staging tree, is now [pushing](<https://lwn.net/Articles/756629/>) to remove Lustre outright: 

While it has been an endless source of enjoyment for new kernel developers learning how to do basic codingstyle cleanups, as well as an semi-entertaining source of bewilderment from the vfs developers any time they have looked into the codebase to try to figure out how to port their latest api changes to this filesystem, it has not really moved forward into the "this is in shape to get out of staging" despite many half-completed attempts. 

Removal from the mainline would, Kroah-Hartman said, allow it to proceed forward at full speed; the project could then return once its code-quality issues have been addressed. 

One of the obvious problems with Lustre is its sheer size; at just under 200,000 lines of code, it's not something that is going to be cleaned up quickly. With that size comes quite a bit of complexity; highly scalable distributed filesystems are not simple, and beginning developers cannot really be expected to make substantive changes to them. 

But the other problem, according to Kroah-Hartman, is that development of Lustre is not actually happening in the staging tree. Instead, the Lustre project maintains its own external tree and makes regular releases outside of the mainline cycle. The [2.11.0 release](<http://lustre.org/lustre-2-11-0-released/>), for example, came out in early April and added a number of new features. Some of the work done in the Lustre repository is sporadically brought over to the copy in the staging tree, but that tree is clearly not the focus of development. As Kroah-Hartman commented: ""This dual-tree development model has never worked, and the state of this codebase is proof of that"". 

Some developers ([Christoph Hellwig](<https://lwn.net/Articles/756631/>), for example) applauded this move. Unsurprisingly, the Lustre developers are somewhat less enthusiastic. Andreas Dilger [argued](<https://lwn.net/Articles/756632/>) that, as a filesystem with thousands of users, its code should be in the mainline (though Kroah-Hartman [countered](<https://lwn.net/Articles/756633/>) that none of those users are running the staging version of the code) and that Lustre has improved considerably over the years. Neil Brown, who has contributed many improvements to Lustre, is also [against its removal](<https://lwn.net/Articles/756634/>), fearing that it would never return afterward. 

What will happen next is unclear. It may be that Kroah-Hartman's real purpose was to light a fire underneath the project and force some action rather than the actual deletion of the code. But there is little doubt that Lustre will eventually find itself staged out if the pace of improvement (and perhaps its development model in general) does not change. Staging is meant to be an entry point into the kernel, not a halfway house where code remains indefinitely.
