---
title: "KS2007: The greater kernel ecosystem and user-space APIs"
url: https://lwn.net/Articles/248376/
date: "September 6, 2007"
category: "Development model-User-space ABI; Sysfs; User-space API"
author: "By Jonathan Corbet September 6, 2007 LWN.net Kernel Summit 2007 coverage"
---

By **Jonathan Corbet**  
September 6, 2007

* * *

LWN.net [Kernel Summit 2007](<https://lwn.net/Articles/KernelSummit2007/>) coverage 

Once upon a time, the kernel exported a small set of system calls which made almost the entire interface with user space. In current times, that interface has grown quite a bit more complex. For all practical purposes, the bottom layer of the system now consists of the kernel plus a fair amount of user-space software - udev, HAL, X, etc., which presents the interface that the user actually sees. A panel at the 2007 Kernel Summit made up of Greg Kroah-Hartman, David Zeuthen, Kay Sievers, and David Airlie looked at issues involving this combined software layer. 

No discussion of the user-space interface is complete without bringing up sysfs and its well-known habit of breaking applications. There are several things being done to minimize sysfs-related problems in the future. The kernel developers have taken a while to learn how to design and manage this interface, and how to represent things in ways that don't break. There is an ongoing effort to break the much-maligned coupling between sysfs and internal kernel data structures. And there is an education effort aimed at helping user-space developers avoid using sysfs in ways which will break in the future. The key here is to bear in mind things can move as the structure of the system changes; they don't necessarily stay put even over a single reboot cycle. Any application which assumes that the system's hardware configuration is stable will break sooner or later. 

Part of this problem is that the dynamic tree structure implemented by sysfs is hard for application developers to work with. The simple, flat namespace found in `/dev` was much easier. User-space developers who don't want to deal directly with sysfs should use the libraries which are part of HAL. The old libsysfs library is gone forevermore; libhal is the new libsysfs. Not everybody in the room agreed that HAL is the way of the future, but that does appear to be the way things are going. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

[Hidden file descriptors](<http://lwn.net/Articles/237722/>) were discussed briefly. Linus said that he thought it was a reasonable idea, but that there have not been a whole lot of developers screaming for that feature. Unless that changes, hidden file descriptors will probably remain outside of the mainline. 

On the X front, much of the work at the moment is aimed at moving video mode setting into the kernel. There are a number of tricky transition issues to take care of; once the kernel is in charge of video modes, it really will not do to have user-space programs changing them behind its back. So kernel-space mode setting will likely remain disabled until the distributor sets a flag indicating that user-space knows not to try to play with the hardware directly. 

There were some questions about how some of the video driver code is managed. This code lives in a repository which provides drivers for both Linux and BSD; there are a certain number of macros in the code designed to make that support easier to maintain. It's a sort of favor being done for the BSD world, and David sees no real need to stop doing that for now. The in-kernel mode setting may force a change, though, as the BSD side is not interested in doing things that way. 

From here, it was a fairly straightforward transition into the next session, which covered review of user-space API additions - system calls in particular. Michael Kerrisk presented an abbreviated version of [his LCE talk on system call review](<http://lwn.net/Articles/247788/>); it was generally received well. 

Christoph Hellwig asked if anybody had reviewed the `timerfd()` and `signalfd()` system calls before they were merged. What followed was one of the few times all day that the room was silent. 

Part of Michael's proposal is that new system calls should come equipped with manual pages. It was suggested that this requirement will be hard to enforce unless the man pages are packaged with the kernel itself. That led to an interesting question: the man pages, as currently written, document the system call interface as presented by the C library. But the API exported directly by the kernel can be different, and often is. Which API should be documented? It seems that the kernel-implemented API is the one to cover, especially considering that glibc is not the only C library and that other library implementors may well be very interested in that information. 

From there the discussion went into the idea of including test cases in the kernel tree as well. In fact, perhaps the entire [Linux Test Project](<http://ltp.sourceforge.net/>) suite could be so packaged. That maybe taking things a little too far, but there was interest in getting a simple set of test cases for new system calls into the kernel. If nothing else, they would help architecture maintainers wire up system calls on their target machines. Christoph Hellwig volunteered to do some of the work to get those tests into the tree, so it might just happen. 

Toward the end, the discussion headed back toward review of new system calls. Linus expressed a fear that an overly severe review process would just force system calls underground (in the form of `ioctl()` commands). No formal decision was made on any sort of review process. But it seems likely that any proposed new system calls will be looked at harder than in the past - at least for a while.
