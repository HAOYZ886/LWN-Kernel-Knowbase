---
title: "rm -r fs/ext3"
url: https://lwn.net/Articles/651645/
date: "July 21, 2015"
category: "Filesystems-ext3"
author: "By Jonathan Corbet July 21, 2015"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 21, 2015

The kernel development community is quite good at adding code to the kernel; its record on removing code is not always quite so bright. There are all kinds of reasons why removing code can be difficult; often, even code that appears to be without use stays around just in case somebody, somewhere, still needs it. Removal can be hard even when there is a known replacement that should work for all users; that can be seen in the case of the ext3 filesystem. 

A few eyebrows went up when Jan Kara [posted a patch](<https://lwn.net/Articles/651072/>) removing the ext3 filesystem recently. Some users clearly thought the move represented a forced upgrade to ext4; Randy Dunlap [remarked that](<https://lwn.net/Articles/651647/>) ""this looks like an April 1 joke to me"". In truth, it is neither a joke nor a forced upgrade; it is, however, an interesting story to look back at. 

Nine years ago, in the middle of 2006, the premier filesystem for most users was ext3, but that filesystem was showing its age in a few ways. Its 32-bit block pointers limited maximum filesystem size to 8TB, a limit that was not too restrictive for most users at the time, but which would be highly problematic today. The filesystem tracks blocks in files with individual pointers, leading to large amounts of metadata overhead and poor performance on larger files. These problems, along with a number of missing features, had long since convinced developers that something newer and better was required. 

For a while, some thought that might be a filesystem called reiser4, but that story failed to work out well even before that filesystem's primary developer left the development community. 

The ext3 developers came up with [a number of patches](<https://lwn.net/Articles/187321/>) aimed at easing its scalability problems. These patches were made directly against the ext3 filesystem, with the idea that ext3 would evolve in the direction that was needed. There was, however, [some resistance](<https://lwn.net/Articles/187336/>) to the idea of making major changes to ext3 from developers who valued that filesystem in its current, stable form. One of those developers, it [turned out](<https://lwn.net/Articles/187349/>), was Linus who, as we all know, has a relatively strong voice in such decisions. 

And so it came to be that the ext3 developers [announced](<https://lwn.net/Articles/189950/>) their intent to create a new filesystem called "ext4"; all new-feature development would be done there. Actually, the new filesystem was first called "ext4dev" to emphasize its experimental nature; the plan was to rename it to "ext4" once things were stable, ""probably in 6-9 months"". In the real world, that [renaming](<http://git.kernel.org/linus/03010a3350301baac2154fa66de925ae2981b7e3>) happened nearly 28 months later and was merged for the 2.6.28 kernel. 

Since then, of course, ext4 has become the primary Linux filesystem for many users. It has seen many new features added, and it is not clear that this process will stop, even though ext4 is now in the same position that ext3 was nine years ago. Through this entire history, though, ext4 has retained the ability to mount and manage ext2 and ext3 filesystems; it can be configured to do so transparently in the absence of the older ext2 and ext3 modules. And, indeed, many distributions now don't bother to build the older filesystem modules, relying on ext4 to manage all three versions of the filesystem. 

Back when ext4 was created, it was envisioned that the older filesystem code would eventually become unnecessary. The plan was that when this happened, ""perhaps 12-18 months out"", the ext3 code would be removed. Once again, reality had something different to say, and the ext3 code endured for over nine years. Unless something surprising happens, though, that record is about to come to an end; ext3 could be removed as soon as the 4.3 development cycle, taking some 28,000 lines of code with it. And most users, even those with ext3 filesystems, will not even notice. 

One might well wonder whether we will see a similar story in the future and the addition of an ext5 filesystem. For the time being, that does not seem to be in the works. Ext4 has picked up a number of features in recent years, with [encryption](<https://lwn.net/Articles/639427/>) as the most recent example, but there has been no talk of moving development to a new source base. Over the years, perhaps, the ext4 developers have done well enough at not breaking things that users are less worried about new development than they once might have been. 

At the other end, there is the question of the ext2 filesystem. That code, too, could be replaced by ext4, but there seems to be no pressure to do so. Ext2 is small, weighing in at less than 10,000 lines of code; ext3 and the associated JBD journaling code come in at 28,000, while ext4 and JBD2 add up to closer to 60,000 lines. The simplicity of ext2 makes it a good filesystem for developers to experiment with, and its maintenance cost is nearly zero. So there is no real reason to take it out anytime soon. 

Ext3, being rather larger than ext2, is a more promising target to remove, though Jan [said](<https://lwn.net/Articles/651649/>) that its maintenance cost was pretty low. The fact that this code has been so thoroughly replaced makes the removal decision relatively easy — but that decision still took nine years to come about. Even so, if all old kernel code were this easy to get rid of, the kernel would be quite a bit smaller than it is today.
