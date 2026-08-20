---
title: In search of the perfect changelog
url: https://lwn.net/Articles/329428/
date: "April 22, 2009"
category: Changelogs; Documentation
author: "By Jonathan Corbet April 22, 2009"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 22, 2009

When kernel developers engage in an extended discussion on the writing of changelogs for patches, one might well conclude that they have run out of useful things to do. But arguments over changelogs are not the same as spelling or grammar flames. In an environment where 10,000 or so changes are merged in every three-month development cycle, developers need all the help they can get to understand what is going into the kernel. Poorly-described patches are harder to understand, and harder to find when searching the history for something specific. So getting changelogs right helps the development process - and the kernel - as a whole. 

It all started innocently enough; Linus was engaging in [a routine patch flaming](<https://lwn.net/Articles/329430/>) when he encountered one of the "Impact:" tags that some developers (especially those working with Ingo Molnar's trees) have adopted in recent months: 

```
Impact: clarify and extend confusing API
```

Suffice to say that he was not much impressed with it: 

And what the hell is up with these bogus "Impact:" things? Who started doing that, and why? If your single-line explanation at the top is not good enough, and your multi-line explanation isn't clear enough, then you should fix the OTHER parts, not add that _idiotic_ "Impact" statement. 

From there, the extended conversation focused on two related topics: the value of "impact" tags and how to write better changelogs in general. On the former, the primary (but not only) proponent of these tags is Ingo Molnar, who [cites several benefits](<https://lwn.net/Articles/329435/>) from their use. Using these tags, he claims, forces developers to write smaller patches which can be adequately described in a single line. They give subsystem maintainers an easy way to assess the changes made by a set of patches and their associated risk; they also make it easier to review a patch against its declared "impact." These tags are also said to force a certain clarity of thought, making developers think through the consequences of a change. 

Most of these arguments leave "Impact:" detractors unmoved, though. Rather than add yet another tag to a patch, they would prefer to see developers just write better changelogs from the outset. In a properly-documented patch, the new tag is just irrelevant. [Andrew Morton said](<https://lwn.net/Articles/329439/>): 

I'm getting quite a few Impact:s now and I must say that the Impact: line is always duplicative of the Subject:. Except in a few cases, and that's because the Subject: sucked. 

Ingo [disputed](<https://lwn.net/Articles/329441/>) that claim at length, needless to say. But he takes things further by stating that, while better changelogs would certainly be desirable, they are not a practical goal. [According to Ingo](<http://lwn.net/Articles/328686/>), most developers are simply not capable of writing good changelogs. Language barriers and such often are part of this problem, but it goes deeper: most developers simply lack the writing skills needed to write clear and concise changelogs. This fact of life, as Ingo sees it, cannot really be changed, but most developers can, at least, be trained to write a reasonable impact tag. 

It is probably fair to say that most developers do not see themselves as being disabled in this way. That said, it is also fair to say that a lot of patches go into the mainline with unhelpful changelogs. That can probably be changed - to an extent at least - through pressure from maintainers and a better understanding of what makes a good changelog. In an attempt to help, your editor has proposed a brief addition to Documentation/development-process: 

Writing good changelogs is a crucial but often-neglected art; it's worth spending another moment discussing this issue. When writing a changelog, you should bear in mind that a number of different people will be reading your words. These include subsystem maintainers and reviewers who need to decide whether the patch should be included, distributors and other maintainers trying to decide whether a patch should be backported to other kernels, bug hunters wondering whether the patch is responsible for a problem they are chasing, users who want to know how the kernel has changed, and more. A good changelog conveys the needed information to all of these people in the most direct and concise way possible. 

To that end, the summary line should describe the effects of and motivation for the change as well as possible given the one-line constraint. The detailed description can then amplify on those topics and provide any needed additional information. If the patch fixes a bug, cite the commit which introduced the bug if possible. If a problem is associated with specific log or compiler output, include that output to help others searching for a solution to the same problem. If the change is meant to support other changes coming in later patch, say so. If internal APIs are changed, detail those changes and how other developers should respond. In general, the more you can put yourself into the shoes of everybody who will be reading your changelog, the better that changelog (and the kernel as a whole) will be. 

Other possible additions have been proposed by [Ted Ts'o](<https://lwn.net/Articles/329443/>) and [Paul Gortmaker](<https://lwn.net/Articles/329444/>). Of course, all of these patches are based on the optimistic notion that developers will actually read the documentation. 

One could argue that the kernel community is rather late in getting around to this kind of discussion. That could be said to be par for the course; in the pre-BitKeeper era (i.e. up to February, 2002), there was almost no tracking of individual changes into the kernel at all. That the fine points of changelogging are being discussed a mere seven years later suggests things are going in the right direction. The level of professionalism in the kernel community has been on the rise for a long time; this process is likely to continue. Whether or not some variant on the impact tag is used in the future, one can assume that the quality of changelogs will, as a whole, be better.
