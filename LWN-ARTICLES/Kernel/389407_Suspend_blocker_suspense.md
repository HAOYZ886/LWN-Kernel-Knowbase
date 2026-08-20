---
title: Suspend blocker suspense
url: https://lwn.net/Articles/389407/
date: "May 26, 2010"
category: "Android; Power management-Opportunistic suspend"
author: "By Jonathan Corbet May 26, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 26, 2010

As of [last week's article](<http://lwn.net/Articles/388131/>) on the Android suspend blocker mechanism, the conversation appeared to be slowing down. Such blessings, it seems, are never permanent; many electrons have been perturbed to continue this discussion since then. The end result is that the late entrance into the discussion by people with names like Alan Cox, Thomas Gleixner, and Peter Zijlstra has made the merging of this feature more unlikely. 

Alan's [dissent](<https://lwn.net/Articles/389409/>) was arguably the most coherent and constructive of just about any that have been posted thus far. He thinks that the problem being addressed by suspend blockers (misbehaving applications) is real, but the solution is wrong. He suggests, instead, the addition of a `setpidle()` system call which indicates the extent to which a process can prevent the system from going into an idle state. If the process is running an untrusted application, the system would be able to go idle (or suspend) even if that process is runnable at the time. More trusted processes (the ones which would be able to use suspend blockers in the Android scheme) would have a higher priority and would be able to run at any time. 

The other piece of the solution, according to Alan, is to put pressure on the authors of badly-written applications. Thomas [agrees](<https://lwn.net/Articles/389411/>): 

We have already proven that the social pressure on crappy applications works. When NOHZ was merged into the kernel we got no effect at all because a big percentage of user space applications just used timers at will and without any thoughts, also it unveiled busy polling and other horrible coding constructs. So what happened ? Arjan created powertop which lets the user analyse the worst offenders in his system. As a result the offending apps got fixed rapidly simply because no maintainer wanted to be on top of the powertop sh*tlist. 

A number of developers have expressed the fear that trying to mitigate the impact of badly-written applications in the kernel will only serve to take the pressure off developers, leading to more bad applications over time. 

Meanwhile, Rafael Wysocki has [sent a pull request](<https://lwn.net/Articles/389412/>) for suspend blockers to Linus, saying: 

Some alternative approaches have been proposed during the on-going discussion, but they are general ideas rather than specific technical propositions, and until someone actually tries to implement them it's not really known whether or not they are suitable for Android. For this reason I don't think we can realistically expect Google to implement any of them. Thus, in my view, we have a choice to either accept or reject this feature altogether. 

As of this writing, Linus has not said what he intends to do. Given the way the conversation has gone, though, it would not be surprising to see the merge window end with no suspend blockers in the mainline. Merging a user-visible feature like suspend blockers is a move which cannot be undone after the 2.6.35 release; when there is this much disagreement, letting another development cycle go by may seem like the prudent thing to do.
