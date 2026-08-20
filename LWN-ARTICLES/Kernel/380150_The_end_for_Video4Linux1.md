---
title: The end for Video4Linux1
url: https://lwn.net/Articles/380150/
date: "March 24, 2010"
category: Video4Linux2
author: "By Jonathan Corbet March 24, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 24, 2010

The Video4Linux1 (V4L1) ABI is deprecated, and has been for a long time; it was ostensibly replaced by Video4Linux2 in the 2.5 development series. But, as has been discovered many times, an ABI is a hard thing to get rid of. So the kernel still supports V4L1 applications; indeed, there are still V4L1-only drivers in current kernels. That situation has persisted for a long time, but it may now be coming to an end. 

Hans Verkuil has posted a multi-stage proposal for the removal of V4L1 from the kernel. [The first phase](<https://lwn.net/Articles/380153/>) involves the conversion of the remaining V4L1 drivers - of which there are several - to the newer ABI. Some of those drivers have since been supplanted by GSPCA and may just be deleted outright. All told, this is a bit of much-needed janitorial work. 

[Phase 2](<https://lwn.net/Articles/380154/>) may be a bit more controversial, though, in that it calls for the removal of the V4L1 compatibility layer in the kernel. This code allows V4L1 applications to work with V4L2 drivers - most of the time. It was an important bit of backward compatibility support, but it has also helped to delay the updating of a number of old V4L1 applications. Given that these applications do still exist (many distributions still ship xawtv, for example), it might be a bit surprising that this layer is slated for removal, perhaps as soon as 2.6.36. 

There are problems with the compatibility layer. It cannot provide access to much of the functionality of contemporary hardware and drivers, it cannot always do the right thing in response to application requests, and it has been a long time since anybody had any interest in maintaining this code. So the V4L developers would like to push it out into user space, and into the libv4l1 library in particular. Supporting old applications would then be a matter of a quick edit (replacing `ioctl()` calls with `v4l1_ioctl()`, for example) and a rebuild against the library. Some old applications may be pulled into the V4L project, since their original maintainers have almost certainly long since lost interest. 

It's not a perfect solution; old, binary applications will cease to work on newer kernels. It is an ABI break, plain and simple, and it is possible that there will be enough of an uproar to prevent this change from happening in the end. But it may also be that nobody really cares about running binary V4L1 applications on new kernels, and that it is truly time for this old interface to pass into history.
