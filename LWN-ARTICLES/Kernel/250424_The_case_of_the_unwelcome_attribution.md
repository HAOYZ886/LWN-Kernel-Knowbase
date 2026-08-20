---
title: The case of the unwelcome attribution
url: https://lwn.net/Articles/250424/
date: "September 19, 2007"
category: Development model
author: "By Jonathan Corbet September 19, 2007"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 19, 2007

A couple of weeks ago, LWN [examined the dispute with the OpenBSD project](<http://lwn.net/Articles/247872/>) over the copyright notices placed in (and removed from) the versions of the Atheros wireless network driver intended for eventual merging into the mainline Linux kernel. At that time, the files with the improperly removed license text had never made it anywhere near the mainline repository and an effort was being made to fix the problem. It really seemed like the whole issue should end then. 

So why does a perusal of the OpenBSD lists (and, often, unfortunately, linux-kernel as well) turn up gems like these? 

The rights and recognition of one of our own developers (reyk@) have been molested, and all we've done as a community is to participate in useless flames and blog postings. Theo has thrown himself, once again, against the spears of the Linux community and their legal vultures in order to protect our software freedoms. How many of us can say we've done our part to defend truly Free Software? 

\-- [Jason Dixon](<https://lwn.net/Articles/250425/>)

In the case of Ryek's [sic] code, the reverse is true but instead of admitting the mistake and making the needed corrections, FSF has pulled out their lawyers in hopes of getting away with the theft. All of this is being done *intentionally* in hopes that no one will put up a fight. 

\-- [J.C. Roberts](<https://lwn.net/Articles/250426/>)

I am really disappointed by all this. I would have expected that once such a patch is suggested (let alone being committed to some public place) some senior/respected/responsible Linux person would tell them what they are doing is wrong. Right from the start. I now see this is not how things work around here. 

\-- [Can E. Acar](<https://lwn.net/Articles/250427/>)

One might well think that the whole issue is still open. In fact, much of the dispute has gone by the wayside. The files with the improperly removed copyright notices never were going to make it to the mainline. The allegations by Theo de Raadt that taking a dual-license notice at its word was illegal have been pretty well laughed off; the OpenBSD camp is no longer asserting that claim. In fact, there is really only one point of dispute left: 

  * The OpenBSD developers do not believe that developers Nick Kossifidis and Jiri Slaby should have added their own copyright attributions to the file `ath5k_hw.c`. Those two developers, it is claimed, have not done enough work on that file to have earned any copyright claims there. 

For this offense, the OpenBSD community continues to flame, threaten lawsuits, and more. It seems that the developers named above should simply add some original haiku to the opening comments so that their right to claim copyright to portions of the file would be indisputable. Even in the absence of bad poetry, these developers have done some small amount of work and will certainly do more to get the code ready for Linux inclusion. Threatening legal action as a way of keeping them from adding their own attribution to the file seems gratuitous. 

Part of what is going on here may be a simple culture clash. It seems that, in the BSD world, the adding of a copyright attribution to a file is usually done with the permission of the existing copyright holders. For a developer to just patch an attribution can come across as being a bit rude. In the Linux community, instead, developers simply add a copyright if they feel they have done enough work to justify it. It is hard to come up with cases where these attributions have gone in without merit. 

Eben Moglen's one [public contribution](<https://lwn.net/Articles/250428/>) to this conversation includes this paragraph: 

We understand that attribution issues are critically important to free software developers; we are accustomed to the strong feelings that are involved in such situations. In the fifteen years I have spent giving free legal help to developers throughout the community, attribution disputes have been, always, the most emotionally charged. 

That is clearly what is going on here - this discussion is certainly happening on a strongly emotional level. But it must be said that the most harsh language seems to be flowing in one direction: from OpenBSD toward Linux. This was also true when [the situation was reversed](<http://lwn.net/Articles/229740/>) and an OpenBSD developer was found to have improperly relicensed some Linux code. In both cases (and in others) there is a clear sense that the OpenBSD people feel wronged by Linux. 

One might well wonder why this is the case. To an extent, OpenBSD developers may be following the tone set by that project's leader. They may be irritated by the licensing asymmetry: BSD-licensed code can be incorporated into a GPL-licensed project, but GPL-licensed code cannot be brought into a BSD-licensed project. Or perhaps they feel that their system has been unfairly upstaged by an inferior rival. Whatever the reason, there is a certain hostility emanating from that camp which is unpleasant to see. 

It would be a mistake, however, to let the public flaming obscure the fact that Linux and the BSD variants have much in common. There is certainly no shortage of Linux proponents whose "advocacy" makes our community look bad. BSD will have people like that too. Meanwhile, behind the scenes, there is a great deal of good will, information, and code which flows in both directions. We are all working toward the same ends, and there are plenty of places where we can learn from the BSD communities. This incident will pass, and hot heads will cool - before, undoubtedly, heating up again on a different topic - but, through it all, free software will just continue to get better.
