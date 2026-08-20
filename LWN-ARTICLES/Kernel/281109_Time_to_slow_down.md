---
title: "Time to slow down?"
url: https://lwn.net/Articles/281109/
date: "May 7, 2008"
category: "Development model-Kernel quality"
author: "By Jonathan Corbet May 7, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 7, 2008

All communities develop rituals over time. One of the enduring linux-kernel rituals is the regular heated discussion on development processes and kernel quality. To an outside observer, these events can give the impression that the whole enterprise is about to come crashing down. But the reality is a lot like the New Year celebrations your editor was privileged enough to see in Beijing: vast amounts of smoke and noise, but everybody gets back to work as usual the next day. 

Beyond that, though, discussions of this nature have real value. Any group which is concerned about issues like quality must, on occasion, take a step back and evaluate the situation. Even if there are no immediate outcomes, the ideas raised often reverberate over the following months, sometimes leading to real improvements. 

The immediate inspiration for this round of discussion was broken systems resulting from the 2.6.26 merge window. This development cycle has had a rougher start than some, with more than the usual number of patches causing boot failures and other sorts of inconvenient behavior. That led to some back-and-forth between developers on how patches should be handled. Broken patches are unfortunate, but one thing is worth noting here: these problems were caught and fixed even before the 2.6.26-rc1 kernel release was made. The problems which set off this round of discussion are not bugs which will affect Linux users. 

But, beyond any doubt, there will be other bugs which are slower to surface and slower to be fixed. The number of these bugs has led to a number of calls to slow down the development process in one way or another. To that end, it is worth noting that the process _has_ slowed down somewhat, with the 2.6.26 merge window bringing in far fewer changesets than were seen for 2.6.24 or 2.6.25. Whether this slower pace will continue into future development cycles, or whether it's simply a lull after two exceptionally busy cycles remains to be seen. 

But, if the process does not slow down on its own, there are developers who would like to find a way to force it to happen. Some have argued for simply throttling the process by, for example, limiting new features in each development cycle to specific subsystems of the kernel. There has also been talk of picking the subsystems with the worst regression counts and excluding new features from those subsystems until things improve. The fact of the matter, though, is that throttling is unlikely to help the situation. 

Slowing down merging does not keep developers from developing, it just keeps their code out of the tree. An extreme example can be found in the 2.4 kernel: the merging of new code was heavily throttled for a long time. What happened was that the distributors started merging new developments themselves because their users were demanding them. So a lot of kernels which went under the name "2.4" were far removed from anything which could be downloaded from kernel.org. That way lies fragmentation - and almost certainly lower quality as well. 

Linus actually [takes this argument](<https://lwn.net/Articles/281122/>) further by arguing that quickly merging patches leads to better quality: 

[M]y personal belief is that the best way to raise quality of code is to distribute it. Yes, as patches for discussion, but even more so as a part of a cohesive whole - as _merged_ patches! 

The thing is, the quality of individual patches isn't what matters! What matters is the quality of the end result. And people are going to be a lot more involved in looking at, testing, and working with code that is merged, rather than code that isn't. 

Andrew Morton has also [argued against throttling](<https://lwn.net/Articles/281125/>): 

If we simply throttled things, people would spend more time watching the shopping channel while merging smaller amounts of the same old crap. 

Kernel developers are, of course, known to be hard-core shoppers, so giving them more opportunity to pursue that activity is probably not the best idea. Seriously, though: Andrew is in favor of a slower development process, but only when approached from a different angle: his point is that an increased focus on quality will, as a side effect, result in slower development. Kernel developers need to be focused on finding and fixing bugs rather than creating new ones and/or shopping. 

It is worth noting that a substantial portion of the development community appears to believe that there are no real problems in this regard. Bugs are being found and fixed at a high rate and the kernel is solid for most users. Arjan van de Ven [notes](<https://lwn.net/Articles/281129/>): 

Are we doing worse on quality? My (subjective) opinion is that we are doing better than last year. We are focused more on quality. We are fixing the bugs that people hit most. We are fixing most of the regressions (yes, not all). Subsystems are seeing flat or lower bugcounts/bugrates. 

Ted Ts'o [points out](<https://lwn.net/Articles/281131/>) that a lot of problems result from obscure and low-quality hardware, and that it's not possible to make everybody happy. Andrew is [unconvinced](<https://lwn.net/Articles/281132/>), though, and seems to fear that the kernel is declining in quality. 

In a sense, though, that part of the discussion is moot. Nobody would argue against the idea that fewer bugs is a worthy goal, regardless of whether one believes that the current process has quality problems. So talk of ways to make things better is always on-topic. 

Testing remains a big issue; the kernel, more than almost any other project, is highly sensitive to the systems on which it is run. Many problems (arguably the majority of them) are related to specific hardware, or specific combinations of hardware; there is no way for the developers, who do not have all possible hardware to test on, to ever find all of these bugs. Users have to help with that process. Getting widespread testing coverage is always hard; Peter Anvin [argues](<https://lwn.net/Articles/281133/>) that the current process has actually made that harder: 

One thing is that we keep fragmenting the tester base by adding new confidence levels: we now have -mm, -next, mainline -git, mainline -rc, mainline release, stable, distro testing, and distro release (and some distros even have aggressive versus conservative tracks.) Furthermore, thanks to craniorectal immersion on the part of graphics vendors, a lot of users have to run proprietary drivers on their "main work" systems, which means they can't even test newer releases even if they would dare. 

There is, in fact, a wealth of development kernels to test, and it is not always clear where users and developers should be concentrating their testing effort. A consensus may be forming, though, that more people should be looking at the linux-next tree in particular. Linux-next is where all of the patches intended for the next merge window are supposed to congregate; the current contents of linux-next, as of this writing, are targeted toward 2.6.27. This is the place where early integration issues and other problems should be found; if linux-next is well tested, the number of problems showing up in the next merge window should be somewhat reduced. 

The linux-next tree is an interesting experiment. It is, for all practical purposes, making the development cycle longer: since linux-next exists, the 2.6.27 cycle has, in some sense, already started. Linux-next also does something which kernel developers have tended to resist: causing the stabilization period for one development cycle to overlap with active development for the next cycle. In the past, it has been argued that this kind of overlap will cause developers to prioritize the creation of new toys over fixing the problems with last week's toys. 

Some people argue that this is happening now: developers are not spending enough time dealing with bugs - and that their carelessness is creating too many bugs in the first place. Others assert that, while it will never be possible to fix every reported bug, the bugs that really matter are being addressed. A real resolution to this disagreement seems unlikely; the creation of meaningful metrics on kernel quality is a difficult task. About the best that can be done is to try to keep the regression list as small as possible; as long as systems which once worked continue to work, it is hard to argue too forcefully that things are headed in the wrong direction.
