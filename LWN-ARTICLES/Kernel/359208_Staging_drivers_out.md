---
title: Staging drivers out
url: https://lwn.net/Articles/359208/
date: "October 28, 2009"
category: Staging tree
author: "By Jonathan Corbet October 28, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 28, 2009

The staging tree was conceived as a way for substandard drivers to get into the kernel tree. Recently, though, there has been talk of using staging to ease drivers out as well. The idea is that apparently unused and unloved drivers would be moved to the staging tree, where they will languish for three development cycles. If nobody has stepped up to maintain those drivers during that time, they will be removed from the tree. This idea was discussed at the 2009 Kernel Summit with no serious dissent. 

Since then, John Linville has decided to test the system with a series of ancient wireless drivers. These include the "strip" driver (""STRIP is a radio protocol developed for the MosquitoNet project - to send Internet traffic using Metricom radios.""), along with the arlan, netwave, and wavelan drivers. Nobody seems to care about this code, and it is unlikely that any users remain. If that is true, then there should be no down side to removing the code. 

That hasn't stopped the complaints, though, mostly from people who believe that staging drivers out of the tree is an abuse of the process which may hurt unsuspecting users. It is true that users may have a hard time noticing this change until the drivers are actually gone - though their distributors may drop them before the mainline does. So the potential for an unpleasant surprise is there; mistaken removals are easily reverted, but that is only partially comforting for a user whose system has just broken. 

The problem here is that there is no other way to get old code out of the tree. Once upon a time, API changes would cause unmaintained code to fail to compile; after an extended period of brokenness, a driver could be safely removed. Contemporary mores require developers to fix all in-tree users of an API they change, though, so this particular indicator no longer exists. That means the tree can fill up with code which is unused and which has long since ceased to work, but which still compiles flawlessly. Somehow a way needs to be found to remove that code. The "staging out" process may not be perfect, but nobody has posted a better idea yet.
