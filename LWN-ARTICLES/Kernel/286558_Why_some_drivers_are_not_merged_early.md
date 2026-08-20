---
title: Why some drivers are not merged early
url: https://lwn.net/Articles/286558/
date: "June 18, 2008"
category: "Development model-Driver merging; Device drivers"
author: "By Jonathan Corbet June 18, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 18, 2008

Arjan van de Ven's kernel oops report always makes for interesting reading; it is a quick summary of what is making the most kernels crash over the past week. It thus points to where some of the most urgent bugs are to be found. Sometimes, though, this report can raise larger issues as well. Consider the [June 16 report](<http://lwn.net/Articles/286520/>), which notes that quite a few kernel crashes were the result of a not-quite-ready wireless update shipped by Fedora. Ingo Molnar was quick to [jump on this report](<https://lwn.net/Articles/286560/>) with a process-related complaint: 

i suspect Fedora has done this to enable more hardware, and/or to fix mainline wireless bugs? I wish we would do such new driver merging in mainline instead, so that we had a single point of testing and single point of effort. 

Same for Nouveau: Fedora carries it and i dont understand why such a major piece of work is not done in mainline and not _helped by_ mainline. 

He then [took the discussion further](<https://lwn.net/Articles/286561/>) with this observation: 

That's my main point: when we mess up and dont merge OSS driver code that was out there in time - and we messed up big time with wireless - we should admit the screwup and swallow the bitter pill. 

This comment drew some unhappy responses from the networking developers, who feel that they have been unfairly targeted for criticism. Wireless drivers have been merged at the first real opportunity, they say, and trying to put them in earlier would have only made things worse. In fact, your editor will submit that mistakes _were_ made with wireless drivers, but those mistakes have little to do with delaying their inclusion into the mainline. What went wrong with wireless is this: 

  * Early wireless developers did not really try to solve the wireless networking problem; they just wanted to get their adaptor to work. Wireless maintainer John Linville once told your editor that, for years, these adaptors were treated as if they were Ethernet adaptors, which they certainly are not. When these developers did get around to dealing with issues specific to wireless networking, they created their own wireless stacks contained within their drivers. So no general wireless framework was created. 

It's only in 2004 that Jeff Garzik [started a project](<http://lwn.net/Articles/87952/>) to create a generic wireless stack for Linux - and he started with a stack (HostAP) which, sometime later on, was seen as not being the best choice. So the work on HostAP - late to begin in the first place \- was eventually abandoned. 

  * The networking stack which was eventually developed - mac80211 - began its life as a proprietary code base created with no community review or oversight at all. Predictably, it had all kinds of problems which required well over a year of work to resolve. Until mac80211 was in reasonable shape, there was no real way to get drivers ready for inclusion. 

The result of all this (and the occasional legal hassle as well) is that wireless networking on Linux lagged for years, and is only now reaching something close to a stable state. So it is not surprising that there has been a lot of code churn in this area, or that things occasionally break. But it is hard to see how trying to merge wireless drivers sooner would have helped the situation significantly. 

The non-merging of the Nouveau driver - the reverse-engineered driver for NVIDIA adapters - also has a simple explanation: the developers have not yet asked for this merge to happen. Nouveau is not considered to be at a point where it works yet, and, importantly, there are still user-space API issues which must be worked out. Breaking user-space code is severely frowned upon, so merging of code is nearly impossible if its user-space interfaces are still in flux. 

James Bottomley [put forward](<https://lists.linux-foundation.org/pipermail/ksummit-2008-discuss/2008-June/000136.html>) another reason why a driver may stay out of the mainline even though the author would like to see it merged: 

For the record, my own view is that when a new driver does appear we have a limited time to get the author to make any necessary changes, so I try to get it reviewed and most of the major issues elucidated as soon as possible. However, since the only leverage I have is inclusion, I tend to hold it out of tree until the problems are sorted out. 

In other words, their control over access to the mainline tree is the one club subsystem maintainers have at hand when they feel the need to push a developer to make changes to a driver. It may well be that simply merging drivers regardless of technical objections (something which a number of developers are pushing for) will reduce the incentive for developers to get their code into top shape - and it's not always clear that others will step in and do the work for them. 

On the other hand, the idea that in-tree code tends to be less buggy than out-of-tree code is relatively uncontroversial. So, for many drivers at least, a "merge first and fix it up later" policy may well lead to the best results in the shortest period of time. One thing that is clear is that this discussion will not be going away anytime soon; chances are good that this year's kernel summit (happening in September) will end up revisiting the issue.
