---
title: "Kernel regression tracking, part 1"
url: https://lwn.net/Articles/737666/
date: "October 31, 2017"
category: Regression tracking
author: "By Jonathan Corbet October 31, 2017 2017 Kernel Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 31, 2017

* * *

[2017 Kernel Summit](<https://lwn.net/Articles/KernelSummit2017/>)

The kernel development community has run for some years without anybody tracking regressions; that changed one year ago when Thorsten Leemhuis stepped up to the task. Two conversations were held on the topic at the 2017 Kernel and Maintainers summits in Prague; this article covers the first of those, held during the open Kernel-Summit track. 

Leemhuis begin by pointing out that he started doing this work even though he does not work for a Linux company; he is, instead, a journalist for the largest computer magazine in Germany. He saw a mention of the gap that was left after Rafael Wysocki stopped [tracking regressions](<https://lwn.net/Articles/414223/>), and thought that he might be a good fit for the job. This work is being done in his spare time. When he started, he had thought that the job would be difficult and frustrating; in reality, it turned out to be even worse than he expected. 

Why is it so hard? The first problem is that nobody actually tells him about regressions, so he has to hunt them down himself. That means digging through a lot of mailing lists and bug trackers. Wysocki noted that things are worse than they were years ago when he did the job, there are a lot more information sources. It is more, Wysocki said, than any one person can follow. 

Leemhuis went on to say that a lot of regressions are also fixed without him even noticing. Nobody tells him about progress toward fixing regressions, so that, too, must be tracked manually. He had [asked developers](<https://lwn.net/Articles/729620/>) to include a special identifier in discussions on regressions, but nobody has done it. That is unfortunate, since he had thought it would be a useful mechanism; perhaps, [![\[Thorsten
Leemhuis\]](https://static.lwn.net/images/conf/2017/osse/ThorstenLeemhuis-sm.jpg)](<https://lwn.net/Articles/737668/>) he said, he should have tried harder. Ben Herrenschmidt agreed, saying that it can be hard to get people to change their established workflow to incorporate a new mechanism. James Bottomley noted that maintainers would, in general, rather avoid having their bugs termed "regressions", since that increases the pressure for an immediate fix. 

Leemhuis raised the idea of creating a dedicated mailing list for regressions, with reporters asked to copy their reports there. Wysocki agreed that this might be useful, but said that the information on how to report regressions properly needs to be better communicated. Laura Abbott concurred, saying that the documentation in this area should be improved. 

Herrenschmidt noted that most bug reports come from distributor kernels rather than the mainline. For distributions like Fedora, which ships something close to a current mainline kernel, these reports can be relevant, though are still a version or two behind the current development kernels. Reports of bugs in enterprise kernels, instead, have little value. Bottomley added that Linus Torvalds is mostly interested in mainline regressions; the resources just don't exist to track regressions in distributor kernels as well. 

There was general agreement that only mainline regressions should be tracked, but Ted Ts'o said that the community could look for volunteers to track regressions in older kernel versions. The work is still useful, he said, and would train others to help with regression tracking. The problem with this idea, Bottomley replied, is that one has to be an idiot to want to do this work — an idea that Leemhuis seemed to concur with. There won't, Bottomley added, be a flood of volunteers in this area. Matthew Wilcox's suggestion that the situation could change because there are a lot of journalists being laid off was not seen as entirely helpful. 

Abbott said that, in her role as a Fedora kernel maintainer, she sees a lot of bug reports, but many of them are of low quality. They need to be filtered before being passed on to any sort of core regression list. Arnd Bergmann added that Linaro has been doing more testing recently and finding regressions in linux-next. But Leemhuis said he is really only interested in regressions that make it to the mainline at this point. 

Leemhuis went on to say that, while Wysocki used the kernel's Bugzilla tracker to handle regressions, it "looks like double-entry accounting" to him and he has avoided it. There is a lot of overhead associated with working in Bugzilla, and kernel developers tend not to like it. So he has been using the mailing lists instead, but perhaps that was the wrong decision? 

Wysocki replied that he used Bugzilla because it was suitable for him; it provided a useful archive of the discussions around regressions. Ts'o said that the real problem is that Torvalds will not dictate a single bug-tracking system for the kernel, so the information is scattered around the community. The kernel Bugzilla is not perfect, he said, but it has the advantage of actually existing and being available. Wysocki added that there needs to be a database somewhere; it should be possible to point people to a definitive entry for a regression. Takashi Iwai said that, for distributors, the most important thing to have is an overview of the situation; that is missing now. There is no comprehensive list of problems, so distributors must go through the time-consuming task of polling a number of different bug trackers. 

Wilcox asked if distributors use the regression list for decisions on which kernel versions to ship, or whether those decisions are purely based on time. Abbott replied that Fedora tries to ship the latest mainline kernel, but the decision on pushing a specific kernel does depend on the current regressions. A significant Intel or AMD graphics regression will cause a kernel to be held back, she said, while "an obscure USB dongle" problem will not. 

Ben Hutchings said that the situation at Debian is similar, at least outside of the long-term support releases. Iwai said that openSUSE Tumbleweed ships the latest kernel, meaning that regression reports are relevant to the current mainline release, not the development kernel that the kernel developers are working on currently. There are, he said, not many people testing the -rc kernels. Jiri Kosina added that SUSE tracks the "Fixes" tags in patches to see which bug fixes are relevant to the kernels they have shipped; those fixes will be backported if needed. That has led to a reduction in the regressions reported with openSUSE kernels. 

Leemhuis asked if he should query developers via email more often the way Wysocki did; Wysocki replied that he didn't do that — his scripts did. Mark Brown said that was a good thing, since the scripts were more polite than their author. Overall, there didn't appear to be any opposition to more email if that's what is needed to improve regression tracking. 

As the discussion came to a close, it was noted that regression reporting is hard for most users. They don't know where to send their reports, and there is little information out there to help them. The noise on the mailing lists does not help. The kernel Bugzilla is especially problematic since it is the wrong place to report many bugs, but it's not clear which ones or where they should actually go. Ts'o said that, if it were up to him, he would designate the Bugzilla as being for all kernel bugs, and that subsystem maintainers would simply be told to cope with it. In the absence of such a policy, users will continue to struggle. 

The final suggestion came from Abbott, who said that perhaps users who send email to the linux-kernel list (and nobody else) should get an automatic response. That response would inform them that email sent only to the list is unlikely to be read by many people and would thus probably not get a response. It would include suggestions regarding how to more successfully report bugs. This idea was generally well received. 

This topic [was revisited during the invitation-only Maintainers Summit](<https://lwn.net/Articles/738216/>) two days later. 

[Your editor would like to thank the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event].
