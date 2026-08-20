---
title: A line in the sand for graphics drivers
url: https://lwn.net/Articles/394702/
date: "July 5, 2010"
category: "Device drivers-Graphics"
author: "By Jonathan Corbet July 5, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 5, 2010

Support for certain classes of hardware has often been problematic for the Linux kernel, and 3D graphics chips have tended to be at the top of the list. Over the last few years, through a combination of openness at Intel and AMD/ATI and reverse engineering for NVIDIA, the graphics problem has mostly been solved - for desktop systems. The situation in the fast-growing mobile space is not so comforting, though. As can be seen in recent conversations, free support for mobile graphics looks like the next big problem to be solved. 

At a first glance, the [announcement](<http://lwn.net/Articles/394665/>) of a 2D/3D driver for Qualcomm "ES 3D" graphics cores (found in the Snapdragon processor which, in turn, is found in a number of high-end smartphones) seems like a good thing. Graphics support for this core is one of the binary blobs which is necessary to run Android on that processor, and it seemed like Qualcomm was saying the right things: 

I'm writing this email because we think it is high time that we get off the bench, into the game and push support for the Qualcomm graphics cores to the mainline kernel. We are looking for advice and comment from the community on the approach we have taken and what steps we might need to take, if any, to modify the driver so it can be accepted. 

Advice and comment is what he got. The problem is that, while the kernel driver is GPL-licensed, it is only a piece of the driver. The code which does the real work of making 3D function on that GPU runs in user space, and it remains locked-down and proprietary. Dave Airlie, the kernel graphics maintainer, [made it quite clear](<https://lwn.net/Articles/394703/>) what he thinks of such drivers: 

We are going to start to see a number of companies in the embedded space submitting 3D drivers for mobile devices to the kernel. I'd like to clarify my position once so they don't all come asking the same questions. 

If you aren't going to create an open userspace driver (either MIT or LGPL) then don't waste time submitting a kernel driver to me. 

Dave's message explains his reasoning in detail; little of it will be new to most LWN readers. He is concerned about possible licensing issues and, at several levels, about the kernel community's ability to verify the driver and to fix it as need be. Dave has also [expressed his resentment](<http://airlied.livejournal.com/73337.html>) on how the mobile chipset vendors are getting great value from Linux but seem to be entirely unwilling to give back to the kernel they have come to depend on so heavily. 

This move may strike some people as surprising. There has been a lot of pressure to get Android-related code into the mainline, but now an important component is being rejected - again. The fact that user-space code is at issue is also significant. The COPYING file shipped with the kernel begins with this text: 

NOTE! This copyright does *not* cover user programs that use kernel services by normal system calls - this is merely considered normal use of the kernel, and does *not* fall under the heading of "derived work". 

Normally, kernel developers see user space as a different world with its own rules; it is not at all common for kernel maintainers to insist on free licensing for user-space components. Dave's raising of licensing issues might also seem, on its face, to run counter to the above text: he is saying explicitly that closed user-space graphics drivers might be a work derived from the kernel and, thus, a violation of the GPL. These claims merit some attention. 

The key text above is "normal system calls." A user-space graphics driver does not communicate with its kernel counterpart with normal system calls; it will use, instead, a set of complex operations which exist only to support that particular chipset. The kernel ABI for graphics drivers is _not_ a narrow or general-purpose interface. The two sides are tightly coupled, a fact which has made the definition of the interface between them into a difficult task - and the maintenance of it almost as hard. While a program using POSIX system calls is clearly not derived from the kernel, the situation with a user-space graphics driver is not nearly so clear. 

It should also be pointed out that, while the kernel community does not normally try to dictate licensing in user space, that community has also never felt bound to add interfaces for the sole use of proprietary code. The resistance to the addition of hooks for anti-malware programs is a classic example. 

But licensing is not the only issue here. In a sense, user-space 3D graphics drivers are really kernel components which simply happen to be running in a separate address space. They necessarily have access to privileged functionality, and they must program a general-purpose processor (the GPU) with the ability to easily hose the system. Without the user-space component, the kernel will not function well. Like other pieces of the kernel, the full 3D driver must be carefully examined to be sure that there are no security problems, fatal bugs, or portability issues. The kernel developers must be able to make changes to the kernel-side driver with full knowledge of what effect those changes will have on the full picture. A proprietary user-space driver clearly makes all of this more difficult - if not impossible. 

User-space binary blob drivers also miss out on many of the important benefits of the free software development process. They will contain bugs and a great deal of duplicated code. 

What Dave (and others) are clearly hoping is that, by pushing back in this way, they will be able to motivate vendors to open up their user-space drivers as well. The history in this regard is encouraging, but mixed. Over time, hardware vendors have generally come to realize that the value they are selling is not in the drivers and that they can make their lives easier by getting their code out into the open. What did it gain all of those wireless networking vendors to implement and maintain their own 802.11 stacks? One can only imagine that they must be glad to be relieved of that burden. But getting them to that point generally required pressure from the kernel development community. 

Hopefully, this pressure will convince at least some mobile 3D vendors to open up as well. That pressure would be increased and made far more effective if at least some device manufacturers would insist on free software support for the components they use. There are companies working in this area which make a lot of noise about their support for Linux. They could do a lot to make Linux better by insisting on that same support from their suppliers. 

Over the years, we have seen that pushing back against binary-only drivers has often resulted in changes for the better; we now have free support for a lot of hardware which was once only supported by proprietary drivers. Some vendors never relent, but enough usually have that the recalcitrant ones can simply be routed around. One shudders to think about what our kernel might look like now had things not gone that way. The prevalence of binary-only drivers in the mobile space shows that this fight is not done, though. 3D graphics drivers are unique in many aspects, including their use of user-space components. But, if we want to have a free kernel in the coming years, we need to hope that they will be subject to the same pressures.
