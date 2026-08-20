---
title: The platform problem
url: https://lwn.net/Articles/443531/
date: "May 18, 2011"
category: Development model
author: "By Jonathan Corbet May 18, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 18, 2011

Your editor first heard the "platform problem" described by Thomas Gleixner. In short, the platform problem comes about when developers view the platform they are developing for as fixed and immutable. These developers feel that the component they are working on specifically (a device driver, say) is the only part that they have any control over. If the kernel somehow makes their job harder, the only alternatives are to avoid or work around it. It is easy to see how such an attitude may come about, but the costs can be high. 

Here is a close-to-home example. Your editor has recently had cause to tear into the cafe_ccic Video4Linux2 driver in order to make it work in settings beyond its original target (which was the OLPC XO 1 laptop). This driver has a fair amount of code for the management of buffers containing image frames: queuing them for data, delivering them to the user, implementing `mmap()`, implementing the various buffer-oriented V4L2 calls, etc. Looking at this code, it is quite clear that it duplicates the functionality provided by the [videobuf](<https://lwn.net/Articles/363349/>) layer. It is hard to imagine what inspired the idiotic cafe_ccic developer to reinvent that particular wheel. 

Or, at least, it would be hard to imagine except for the inconvenient fact that said idiotic developer is, yes, your editor. The reasoning at the time was simple: videobuf assumed that the underlying device was able to perform scatter/gather DMA operations; the Cafe device was nowhere near so enlightened. The obvious right thing to do was to extend videobuf to handle devices which were limited to contiguous DMA operations; this job was eventually done by Magnus Damm a couple years later. But, for the purposes of getting the cafe_ccic driver going, it simply seemed quicker and easier to implement the needed functionality inside the driver itself. 

That decision had a cost beyond the bloating of the driver and the kernel as a whole. Who knows how many other drivers might have benefited from the missing capability in the years before it was finally implemented? An opportunity to better understand (and improve) an important support layer was passed up. As videobuf has improved over the years, the cafe_ccic driver has been stuck with its own, internal implementation which has seen no improvements at all. We ended up with a dead-end, one-off solution instead of a feature that would have been more widely useful. 

Clearly, with hindsight, the decision not to improve videobuf was a mistake. In truth, it wasn't even a proper decision; that option was never really considered as a way to solve the problem. Videobuf could not solve the problem at hand, so it was simply eliminated from consideration. The sad fact is that this kind of thinking is rampant in the kernel community - and well beyond. The platform for which a piece of code is being written appears fixed and not amenable to change. 

It is not all that hard to see how this kind of mindset can come about. When one develops for a proprietary operating system, the platform is indeed fixed. Many developers have gone through periods of their career where the only alternative was to work around whatever obnoxiousness the target platform might present. It doesn't help that certain layers of the free software stack also [seem frustratingly unfixable](<http://airlied.livejournal.com/74826.html>) to those who have to deal with them. Much of the time, there appears to be no alternative to coping with whatever has been provided. 

But the truth of the matter is that we have, over the course of many years, managed to create a free operating system for ourselves. That freedom brings many advantages, including the ability to reach across arbitrary module boundaries and fix problems encountered in other parts of the system. We don't have to put up with bugs or inadequate features in the code we use; we can make it work properly instead. That is a valuable freedom that we do not exploit to its fullest. 

This is a hard lesson to teach to developers, though. A driver developer with limited time does not want to be told that a bunch of duplicated or workaround code should be deleted and common code improved instead. Indeed, at a kernel summit a few years ago, it was generally agreed that, while such fixes could be requested of developers, to require them as a condition for the merging of a patch was not reasonable. While we can encourage developers to think outside of their specific project, we cannot normally require them to do so. 

Beyond that, working on common code can be challenging and intimidating. It may force a developer to move out of his or her comfort zone. Changes to common code tend to attract more attention and are often held to higher standards. There is always the potential of breaking other users of that code. There may simply be the lack of time for - or interest in - developing the wider view of the system which is needed for successful development of common code. 

There are no simple solutions to the platform problem. A lot of it comes down to oversight and mentoring; see, for example, the ongoing effort to improve the ARM tree, which has a severe case of this problem. Developers who have supported the idea of bringing [more projects together in the same repository](<https://lwn.net/Articles/417647/>) also have the platform problem in mind; their goal is to make the lines between projects softer and easier to cross. But, given how often this problem shows up just within the kernel, it's clear that separate repositories are not really the problem. What's really needed is for developers to understand at a deep level that platforms are amenable to change and that one does not have to live with second-rate support.
