---
title: Debugfs and the making of a stable ABI
url: https://lwn.net/Articles/309298/
date: "December 3, 2008"
category: "Development model-User-space ABI"
author: "By Jonathan Corbet December 3, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
December 3, 2008

Remi Colinet recently [proposed](<https://lwn.net/Articles/309299/>) the addition of a new virtual file, `/proc/mempool`, which would display the usage of memory pools within the kernel. Nobody really disagreed with the idea of making this information available, but there were some grumbles about putting it into `/proc`. Once upon a time, just about anything could go into that directory, but, in recent years, there has been a real attempt to confine `/proc` to its original intent: providing information about processes. `/proc/mempool` is not about processes, so it was considered procfile-non-grata. It was suggested that another home should be found for this file. 

Where that other home should be is not obvious, though. Somewhere like `/sys/kernel` might seem to make sense, but sysfs has rules of its own. In particular, the one-value-per-file rule makes it hard to create an easy file where developers can simply query the state of a kernel subsystem, so sysfs is not a suitable home for this file either. 

The next option is debugfs, which was [created](<http://lwn.net/Articles/115405/>) in December, 2004. Debugfs is meant to be an aid for kernel developers; it explicitly disclaims any rules on the types of files that can be put there. All rules except for one: debugfs is not a mandatory part of any kernel installation, and nothing found therein should be considered to be a part of the stable user-space ABI. It is, instead, a dumping ground where kernel developers can quickly export information which is useful to them. 

Since debugfs is not a part of the user-space ABI, it seems like a poor place to put things that users might depend on. When this was pointed out, it became clear that the non-ABI status of debugfs is not as well established as one might think. [Quoting Matt Mackall](<https://lwn.net/Articles/309301/>): 

The problem with debugfs is that it claims to not be an ABI but it is lying. Distributions ship tools that depend on portions of debugfs. And they also ship debugfs in their kernel. So it is effectively the same as /proc, except with the 1.0-era everything-goes attitude rather than the 2.6-era we-should-really-think-about-this one. 

Pushing stuff from procfs to debugfs is thus just setting us up for pain down the road. Don't do it. In five years, we'll discover we can't turn debugfs off or even clean it up because too much relies on it. 

As an example, Matt pointed out the [extensively-documented usbmon interface](<https://lwn.net/Articles/309302/>) which provides a great deal of information about what's happening on a USB bus. If it is not an ABI, he says, nobody should be upset if he submits a patch which breaks it. 

That is a perennial problem with interfaces between the kernel and user space; changing them causes pain for users. That is why incompatible changes to user-space interfaces are almost never allowed; an important goal for the kernel development process is to avoid breaking user-space programs. One might think that this problem could be avoided for a specific interface by explicitly documenting it as an unstable interface. The files in `Documentation/ABI/testing` are meant to serve that role; anything found there should be considered to be unstable. But, as soon as people start using programs which depend on a specific interface, it has, for all practical purposes, hardened into part of the kernel ABI. 

Linus [put it this way](<https://lwn.net/Articles/309305/>): 

The fact that something is documented (whether correctly or not) has absolutely _zero_ impact on anything at all. What makes something an ABI is that it's useful and available. The only way something isn't an ABI is by _explicitly_ making sure that it's not available even by mistake in a stable form for binary use. 

Example: kernel internal data structures and function calls. We make sure that you simply _cannot_ make a binary that works across kernel versions. That is the only way for an ABI to not form. 

So a given kernel interface can be kept away from ABI status if it is so hard to get to, and so unstable, that nothing ever comes to depend on it. The kernel module interface certainly fits this bill. Modules must generally be built for the exact kernel they are intended to work with, and they must often be built with the same configuration options and the same compiler. Anybody who has gotten into the dark business of distributing binary-only modules has learned what a challenge it can be. 

Debugfs is different, though. It is enabled in a number of distributor kernels, even if, perhaps, it is not mounted by default. Once a set of files gets placed there, their format tends to change rarely. So it is possible for people to write programs which depend on debugfs files. And the end result of that is that debugfs files can become part of the stable kernel ABI. That is generally not a result that was intended by anybody involved, but it happens anyway. The only way to avoid it would be to deliberately shake up debugfs every kernel cycle - and few developers have much desire to do that. 

This is a discussion without a whole lot in the way of useful conclusions; it leaves `/proc/mempool` without a home. ABI design, it turns out, is still hard. In the longer term, dealing with an ABI which was never really designed, but which just sort of settled into being, is even harder. There does not appear to be any substitute for thinking seriously about every interface between kernel and user space, even if it's just for a developer's debugging tool.
