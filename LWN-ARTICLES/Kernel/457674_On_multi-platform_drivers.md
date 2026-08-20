---
title: "On multi-platform drivers"
url: https://lwn.net/Articles/457674/
date: "September 7, 2011"
category: Development model
author: "By Jonathan Corbet September 7, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 7, 2011

LWN recently [looked at the discussion around moving the Broadcom wireless driver](<https://lwn.net/Articles/456762/>) into the mainline from the staging tree. This driver raises a number of issues on how the kernel community interacts with hardware manufacturers. One important aspect of the discussion, though, did not come up until after that article was written. Linux drivers are expected to be drivers for Linux, only. Attempts to maintain a Linux driver as a multi-platform driver will lead to unhappiness, for a number of reasons. What follows is an unabashedly partisan article on why multi-platform drivers do not fit well with the Linux kernel. 

Broadcom developer Henry Ptasinski [brought the issue to the fore](<https://lwn.net/Articles/456872/>) while talking about why the company was not interested in supporting the in-mainline b43 driver: 

The brcmsmac driver has architectural alignment with our drivers for other operating systems, and we intend to to enhance and maintain this driver in parallel with drivers for other operating systems. Maintaining alignment between our Linux driver and drivers for other operating systems allows us to leverage feature and chip support across all platforms. 

To developers who have worked with the kernel for a while, these words look like a fundamental mistake. To others, though, they seem reasonable: if Broadcom wants to support its hardware, why not let the company do things its way? 

One clear problem with trying to maintain "architectural alignment" with drivers for other operating systems is that only the original company can maintain that alignment. The other associated drivers are almost certainly not open source; nobody else in the community has any way to know which changes are consistent with those other drivers and which are not. Not even the relevant subsystem maintainer can make that kind of call. 

One also must consider that most other kernel developers have no motivation for - or interest in - maintaining the correspondence between the drivers, even if they did know how to do it. 

The obvious conclusion here is that allowing a vendor to maintain a multi-platform driver in the kernel tree will only work if that vendor is given absolute control over the code. If others can make arbitrary changes, there is no way for the vendor to keep the drivers consistent. But, in the kernel, nobody has that kind of absolute control with the possible exception of Linus Torvalds. If something needs to be fixed or changed, anybody with the relevant technical skills can do it. If a piece of the kernel tree were to be fenced off and made off-limits for kernel developers, the kernel as a whole becomes a little less free. 

And that freedom matters. Consider the problem of internal API changes. As anybody who watches kernel development knows, internal interfaces are changed all the time in response to problems and changing needs. Those changes can, at times, force significant changes in users of the affected interfaces. Contemporary rules call for a developer who makes an interface change to fix any code broken by that change. Code that has been designated as off limits will be hard to fix in this way, slowing down the evolution of the kernel as a whole. As one example, consider the removal of the big kernel lock; that job required significant locking changes in many places. Literally hundreds of drivers were modified in the process. Impeding those changes would have made the BKL removal task even slower - and maybe impossible. 

Manufacturers are not known for long-term support of their products; they have no real reason to support five-year-old chipsets that they no longer sell. Indeed, they have every reason to end such support and encourage the replacement of older hardware with shiny new stuff. Linux, instead, tends to support hardware for as long as it is in use. Giving a vendor absolute control over a driver is certain to create conflict when that vendor moves to drop support for older chipsets. 

A vendor's agenda can differ from the community's needs in other ways as well. Vendors may not appreciate patches to enable undocumented features or make low-end offerings behave more like their more expensive alternatives. Or consider Hans Reiser's [opposition](<https://lwn.net/Articles/82480/>) to the addition of extended attribute and access control list support to reiserfs. His argument was that users should wait for the shiny new Reiser4 filesystem to obtain such features; had he been listened to, reiserfs users never would have had support for those basic filesystem capabilities. The kernel works well because it is maintained as the best kernel for all users over the long term, even if that occasionally causes conflicts with short-term vendor desires. 

Multi-platform drivers from vendors tend to be written around the minimal set of support functions that are available on all platforms. The result is a lot of code duplicating functionality already found in the Linux kernel; consider, for example, just how many wireless drivers initially came with their own 802.11 stacks built in. Developing and maintaining just one rock-solid 802.11 implementation is hard; having several of them in the kernel would bloat the kernel and endow it with a number of second-rate implementations - all of which must be maintained into the future. Other kernel support code - from simple linked lists through complicated memory management - is also often avoided by multi-platform drivers. Those drivers will be bigger, buggier, and harder for kernel developers to read and support. They are also much less likely to behave consistently with other Linux drivers for the same type of hardware. 

Beyond all of the above, it is also far from clear that maintaining a multi-platform driver actually saves any work. Drivers written for Linux can make full use of the available support infrastructure. Multi-platform drivers must, instead, duplicate much of that functionality and maintain an operating system abstraction layer as well. Maintaining a multi-platform driver means maintaining a larger body of code without help from the community. 

In summary: trying to maintain a single driver for multiple operating systems may look like a good idea on the surface. But it is only sustainable in a world where the vendor keeps complete control over the code. Even then, it leads to worse code, duplicated effort, long-term maintenance issues, and more work overall. Linux works best when its drivers are written for Linux and can be fully integrated with the rest of the kernel. The community's developers understand this well; that is why multi-platform drivers have a hard time getting into the mainline.
