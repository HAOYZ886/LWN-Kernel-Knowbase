---
title: "A Linux kernel compatibility layer for FreeBSD?"
url: https://lwn.net/Articles/421601/
date: "January 5, 2011"
category: InfiniBand
author: "By Jake Edge January 5, 2011"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
January 5, 2011

As part of his work in porting a Linux-based InfiniBand stack to FreeBSD, Jeff Roberson put together a compatibility layer that maps Linux-specific kernel functionality to FreeBSD equivalents. He [announced](<https://lwn.net/Articles/421605/>) this effort on the freebsd-arch mailing list in order to get a general sense of what other FreeBSD developers thought of the idea--and whether it might make sense to combine it with other, similar, compatibility shims. The thread gives an interesting look into the problems that less popular operating systems have when trying to support new functionality that is already well-established in Linux, even if all of that code is [not in the mainline](<https://lwn.net/Articles/412748/>). 

The InfiniBand stack that Roberson ported was developed by the [OpenFabrics Alliance](<http://www.openfabrics.org/>) as part of its OpenFabrics Enterprise Distribution (OFED). It largely consists of patches on top of the existing Linux InfiniBand code. All of the code is available under both the GPL and BSD licenses, which is what allowed Roberson to port it to FreeBSD. In order to minimize changes to that stack, though, he created a fairly large compatibility layer: 

The infiniband port has been done by creating a 10,000 line KPI [Kernel Programming Interface] compatability layer. With this layer the vast majority of the driver code runs unmodified. The exceptions are anything that interfaces with skbs and most of the code that deals with network interfaces. 

Roberson goes on to list the kinds of things that the compatibility layer supports, including atomic operations, various device types, interrupts, kobjects, radix and rbtrees, lists, spinlocks, and so on. For the most part, it is just translating Linux-isms into their FreeBSD counterparts. The current state of the wrapper can be seen in Roberson's [svn repository](<http://svn.freebsd.org/viewvc/base/projects/ofed/head/sys/ofed/include/linux/>), which consists of a linux kernel include tree along with code to implement those Linux-isms. The OFED code can then be built by pointing it at the include tree. 

Only those parts of Linux that were needed by the OFED code were actually implemented in the compatibility layer, but Roberson wonders if it makes sense to combine his effort with others: 

I have seen that some attempt at similar wrappers has been made elsewhere. I wonder if instead of making each one tailored to individual components, which mostly seem to be filesystems so far, should we put this in a central place under compat somewhere? 

While there was some predictable Linux-bashing in the thread, most participants seemed favorably inclined toward the compatibility layer itself, as well as making it more widely available to other parts of the FreeBSD kernel tree. There are some difficulties, of course, including exactly _which_ Linux kernel version should be tracked. As was [pointed out](<https://lwn.net/Articles/421612/>), somewhat caustically, by Garrett Wollman in the thread, internal Linux interfaces change with some frequency, and without much warning: 

Fundamentally, maintaining any sort of Linux compatibility is a losing battle, since the hordes will keep on rototilling interfaces in every release until the cows come home, with no concern (and in many cases utter contempt) for anyone else who might need to maintain kernel code. It's a testament to their size and ability that they have managed to keep the system relatively usable and stable over the long term when major parts of the system get replaced on such a regular basis. 

It's not just the size and ability of the Linux community that has led to a stable and usable system, though. As Matthew Jacob [notes](<https://lwn.net/Articles/421672/>), Linux developers point to the ability to change the internal interfaces as one of the strengths of their development model. Although the FreeBSD developers may not agree with it, the [stable_api_nonsense](<http://www.kernel.org/doc/Documentation/stable_api_nonsense.txt>) document does provide reasons for the Linux developers' disdain for stable internal APIs. 

Still, choosing a kernel version to track is not necessarily easy. Based on some previous work maintaining an out-of-tree Linux filesystem, Chris BeHanna was able to [offer some advice](<https://lwn.net/Articles/421618/>) on that: 

I would say that you want to pick a major release of one or at most two major commercial distros (I'll pick on Red Hat Enterprise Linux and the enterprise SuSE for illustration), and track the major releases of them that correspond to roughly the same Linux kernel version, and then STICK to that for the life of a given FreeBSD release stream. Commercial vendors backport fixes into their otherwise-fairly-stable major releases, which minimizes your pain. There are still some changes here and there along the way, and you have to decide how many kernel config options you are willing to support (I'd suggest the default SMP, maybe the default PAE, and punt everything else or you'll go mad). 

The combinatorial pain of committing to more than that leads to a desire for seppuku: I had to support multiple kernel versions each (and often multiple configurations per kernel) for Red Hat 7, 8, 9; RHEL 4 and 5, SuSE Enterprise 9, 10, 10.1, and 10.2, SuSE Desktop 9 and 10, and Fedora 4, 5, and 6 across x86, x86-64, and IA-64, and it ended up being more than 400 binaries to be generated, tested, and delivered every time we issued a patch release. 

There are some obvious advantages to using the OFED code with minimal modifications. Bug fixes and enhancements are being made regularly that could be incorporated into the FreeBSD kernel fairly easily using the compatibility layer. But some argue that adding such a layer will result in fewer vendors trying to get their code working natively for FreeBSD, likening it to the [linuxulator](<http://www.leidinger.net/blog/2010/09/28/the-freebsd-linuxulator-explained-for-users/>) that allows Linux binaries to run on FreeBSD. Alexander Kabaev [put it](<https://lwn.net/Articles/421629/>) this way: 

[...] [A] Linux compat layer in kernel will have the same effect as linuxulator in userland - we do have some vendors still trying to bother with FreeBSD drivers for their hardware now and we will have none after we provide the possibility to hack their Linux code to run somewhat stably on top of Linux compat layer. 

But Roberson is [not convinced](<https://lwn.net/Articles/421630/>) that the linuxulator has had a negative effect on FreeBSD market share, and instead sees it as allowing FreeBSD to ""stay relevant in areas where companies can not justify an independent FreeBSD effort. Adobe is a good example of this."" 

Roberson seems to be taking a pragmatic approach to getting InfiniBand support into current versions of FreeBSD. Evidently there is an InfiniBand stack that runs on earlier versions of FreeBSD, but it hasn't been maintained for more recent kernels. There aren't any real alternatives, as he sees it: 

Our options are, to leave FreeBSD users without infiniband, which I can tell you has cost us more market share as I know of specific cases we have lost due to it. To maintain our own stack independently, which no one has the budget for. Or to try to integrate with OFED. Do you see some other approach? 

So far, there has been no answer to his question. It may be that the compatibility layer continues to live in the InfiniBand tree, rather than move to a more widely accessible place, but there seems to be no one stepping up to develop and maintain a FreeBSD-specific stack. While there is some resistance to the 10,000-line size of the layer, it is, for the most part, a pretty thin shim, as Roberson describes: 

Let's talk nuts and bolts about what this thing does. In the vast majority of cases it simply shuffles arguments and function names around where there is a 1:1 correlation between linux api and FreeBSD API. Think about things like atomics, callouts, locks, jiffies vs ticks, etc. In these areas the systems are trivially different. In a very small number of areas where this wasn't the case I did a direct port and noted it with an #ifdef. 

It wasn't brought up in the thread, but there may also be questions about the license of the compatibility layer (seemingly BSD-only) at some point. Lawyers might argue that it is a derivative work of the Linux kernel. It's unlikely to be a real problem unless someone takes the code into a closed-source product, which is something that the BSD license allows. Specifically licensing it under the GPL would avoid that particular problem, though that might not fit well with the goals of the FreeBSD project. 

It is undoubtedly galling to some in the FreeBSD community to adopt a big chunk of Linux code along with a Linux mapping layer, but the reality is that the community isn't large enough to go it alone, at least for InfiniBand. In some ways it is reminiscent of the [problems](<https://lwn.net/Articles/296444/>) that came about when the new direct rendering manager (DRM) development process was announced. In both cases, the community of FreeBSD users and developers is just not large enough to justify some kind of dual-OS approach. 

On the other hand, though, in both cases the licenses specifically allow Linux and FreeBSD (and others) to incorporate the code. While there are lots of technical and development process barriers to more collaboration between Linux and the BSD world, the usually painful license incompatibility issue did not rear its head. With luck, that will play out well for both sides, with dual-licensed patches going in both directions, hopefully without the [rancor](<https://lwn.net/Articles/250424/>) that sometimes occurs. 

[Thanks to James Andrewartha for giving us a heads-up about this topic.]
