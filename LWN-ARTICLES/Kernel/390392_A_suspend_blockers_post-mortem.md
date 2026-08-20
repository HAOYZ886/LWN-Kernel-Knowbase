---
title: "A suspend blockers post-mortem"
url: https://lwn.net/Articles/390392/
date: "June 2, 2010"
category: Development model
author: "By Jonathan Corbet June 2, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
June 2, 2010

The failure of the lengthy attempt to get Android's suspend blockers patch set into the kernel offers a number of lessons at various levels. The technical side of this episode has been covered in [a Kernel-page article](<https://lwn.net/Articles/390369/>); here we'll look, instead, at the process issues. Your editor argues that this sequence of events shows both the best and the worst of how Linux kernel development is done. With luck, we'll learn from both and try to show more of the best in the future. 

Suspend blockers first surfaced as [wakelocks](<http://lwn.net/Articles/318611/>) in February, 2009. They were immediately and roundly criticized by the development community; in response, Android developer Arve Hjønnevåg made a long series of changes before eventually bowing to product schedules and letting the patches drop for some months. After the Linux Foundation's Collaboration Summit this year, Arve came back with a new version of the patch set after being encouraged to do so by a number of developers. Several rounds of revisions later, each seemingly driven by a new set of developers who came in with new complaints, these patches failed to get into the mainline and, at this point, probably never will. 

In a number of ways, the situation looks pretty grim - an expensive failure of the kernel development process. Ted Ts'o [described it this way](<https://lwn.net/Articles/390378/>): 

Keep in mind how this looks from an outsider's perspective; an embedded manufacturer spends a fairly large amount of time answering one set of review comments, and a few weeks later, more people pop up, and make a much deeper set of complaints, and request that the current implementation be completely thrown out and that someone new be designed from scratch --- and the new design isn't even going to meet all of the requirements that the embedded manufacturer thought were necessary. Is it any wonder a number of embedded developers have decided it's Just Too Hard to Work With the LKML? 

Ted's comments point to what is arguably the most discouraging part of the suspend blocker story: the Android developers were given conflicting advice over the course of more than one year. They were told several times: fix X to get this code merged. But once they had fixed X, another group of developers came along and insisted that they fix Y instead. There never seemed to be a point where the job was done - the finish line kept moving every time they seemed to get close to it. The developers who had the most say in the matter did not, for the most part, weigh in until the last week or so, when they decisively killed any chance of this code being merged. 

Meanwhile, in public, the Android developers were being criticized for not getting their code upstream and having their code removed from the staging tree. It can only have been demoralizing - and [expensive too](<https://lwn.net/Articles/390397/>): 

At this point we've spent more engineering time on revising this one patchset (10 revisions to address various rounds of feedback) and discussion of it than we have on rebasing our working kernel trees to roughly every other linux release from 2.6.16 to 2.6.32 (which became much easier after switching to git). 

No doubt plenty of others would have long since given up and walked away. 

There are plenty of criticisms which can be directed against Android, starting with the way they developed a short-term solution behind closed doors and shipped it in thousands of handsets before even trying to upstream the code. That is not the way the "upstream first" policy says things should be done; that policy is there to prevent just this sort of episode. Once the code has been shipped and applications depend on it, making any sort of change becomes much harder. 

On the other hand, it clearly would not have been reasonable to expect the Android project to delay the shipping of handsets for well over a year while the kernel community argued about suspend blockers. 

In any case, this should be noted: once the Android developers decided to engage with the kernel community, they did so in a persistent, professional, and solution-oriented manner. They deserve some real credit for trying to do the right thing, even when "the right thing" looks like a different solution than the one they implemented. 

The development community can also certainly be criticized for allowing this situation to go on for so long before coming together and working out a mutually acceptable solution. It is hard to say, though, how we could have done better. While kernel developers often see defending the quality of the kernel as a whole as part of their jobs, it's hard to tell them that helping others find the right solutions to problems is also a part of their jobs. Kernel developers tend to be busy people. So, while it is unfortunate that so many of them did not jump in until motivated by the imminent merging of the suspend blocker code, it's also an entirely understandable expression of basic human nature. 

Anybody who wants to criticize the process needs to look at one other thing: in the end it appears to have come out with a better solution. Suspend blockers work well for current Android phones, but they are a special-case solution which will not work well for other use cases, and might not even work well on future Android-based hardware. The proposed alternative, based on a quality-of-service mechanism, seems likely to be more flexible, more maintainable, and better applicable to other situations (including realtime and virtualization). Had suspend blockers been accepted, it would have been that much harder to implement the better solution later on. 

And that points to how one of the best aspects of the kernel development process was on display here as well. We don't accept solutions which look like they may not stand the test of time, and we don't accept code just because it is already in wide use. That has a lot to do with how we've managed to keep the code base vital and maintainable through nearly twenty years of active development. Without that kind of discipline, the kernel would have long since collapsed under its own weight. So, while we can certainly try to find ways to make the contribution process less painful in situations like this, we cannot compromise on code quality and maintainability. After all, we fully expect to still be running (and developing) Linux-based systems after another twenty years.
