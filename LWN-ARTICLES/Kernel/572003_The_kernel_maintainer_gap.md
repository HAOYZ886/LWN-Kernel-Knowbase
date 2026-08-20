---
title: The kernel maintainer gap
url: https://lwn.net/Articles/572003/
date: "October 30, 2013"
category: "Development model-Maintainers"
author: "By Jake Edge October 30, 2013 ELC Europe"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
October 30, 2013

* * *

[ELC Europe](<https://lwn.net/Archives/ConferenceByYear/#2013-Embedded_Linux_Conference_Europe>)

Wolfram Sang is worried that the number of kernel maintainers is not scaling with the number of patches flowing into the mainline. He has collected some statistics to quantify the problem and he reported those findings at the 2013 [Embedded Linux Conference Europe](<http://events.linuxfoundation.org/events/embedded-linux-conference-europe>). There is not an imminent collapse in the cards, according to his data, but he does show that the problem is already present and he forecasts that it will only get worse. 

[ ![\[Wolfram Sang\]](https://static.lwn.net/images/2013/elce-sang-sm.jpg) ](<https://lwn.net/Articles/572040/>)

Sang's [slides [PDF]](<http://events.linuxfoundation.org/sites/events/files/slides/ELCE2013_WolframSang_WeHaveAScalingProblem.pdf>) had numerous graphs (some of which are reproduced here). The first simply showed the number of patches that went into each kernel from 3.0 to 3.10. As one would guess, the trend is an increase in the number of patches. Companies are working more with the upstream kernel than they have in the past, he said, which is great, but leads to more patches. 

There are also, unsurprisingly, more contributors. Using the tags in Git commits, Sang counted the number of authors and contrasted that with the number of "reviewers" (using the "Committer" tag) in his second graph. Over the 3.0–3.10 period, the number of authors rose by around 200, which is, coincidentally, roughly the (largely static) number of reviewers. That means that the gap between authors and reviewers is getting larger over time, which is a basic outline of the scaling problem, he said. If we want to maintain Linux with the quality we have come to expect, it is a problem we need to pay attention to. 

His statistics are based on accepted patches and don't include superseded patches or bogus patches that might require a lengthy explanation to the author. There is also a fair amount of education that maintainers often do for new developers. All of that takes additional time beyond what a raw number patches will show, he said. While his graphs start at 3.0, he does not want to give the impression that the maintainer workload was in a good state at that time, "it was challenging already" and it is getting worse. 

Trond Myklebust gave the best [definition of a maintainer](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000066.html>) that Sang knows of. According to that definition, the job is made of five separate roles: software architect, software developer, patch reviewer, patch committer, and software maintainer. It is not easy to rip out any of those tasks to distribute them to other people. The "number one rule is to get more maintainers who like to do all of these jobs at once". 

[ ![\[Authors vs. others graph\]](https://static.lwn.net/images/2013/elce-people-stats-sm.png) ](<https://lwn.net/Articles/572041/>)

The right maintainer will enjoy all of those jobs, which makes good candidates fairly rare. Sang suggested that developers remember that maintainers have all those roles when interacting with them. He doesn't mean that developers should obey maintainers all the time, he said, but they should keep in mind that the maintainer may be wearing their architect hat so they may be looking beyond the direct problem the developer is trying to solve. 

The "Reviewed-By" and "Tested-By" tags are quite helpful to him as a maintainer because they indicate that the patch is useful to others beyond just its author. That led him to look at the stats for those tags. He plotted the number of reviewers and testers who were not also committers to try to gauge that pool. That graph appears above at right, and shows that there are around 200 reviewers and 200 testers for each kernel cycle as well. The trend is much like that of maintainers, so there is still an increasing gap. The reviewers and testers "are doing great work", but more of them are needed as well. 

Using a diagram of the different pieces in a typical ARM system on chip (SoC), Sang showed that there are many different subsystems that go into a kernel for a particular SoC. He wanted to look at those subsystems to see how well they are functioning in terms of how quickly patches are being merged. He also wanted to compare the i2c subsystem that he maintains to see how it measured up. 

[ ![\[Subsystem latency graph\]](https://static.lwn.net/images/2013/elce-latency-all-sm.png) ](<https://lwn.net/Articles/572042/>)

Using the "AuthorDate" and "CommitDate" tags on patches from 3.0 to 3.10, he measured the latency of patches for several different subsystems (the combined graph is shown at right). That metric can be inaccurate as a true measure of latency if there are a lot of merge commits (as the CommitDate may not reflect when the maintainer added the patch), but that was not really a problem for the subsystems he looked at, he said. 

He started with the `drivers/net/ethernet` subsystem, which had some 5000 patches over the kernel releases measured. It has a fairly low latency, with 85% of the patches being merged within 28 days. This is what developers want, he said, a prompt response to their patches. In fact, 70% of patches were merged within one week for that subsystem. 

Looking at the `mtd` subsystem shows a different story. Sang was careful to point out that he was not "bashing" any subsystem or maintainer as they all do "great work" and do what they can to maintain their trees. After 28 days, `mtd` had merged just over 50% of its patches. Those might be more complicated patches so they take more review time, he said. That is difficult to measure. After one three-month kernel cycle, about 80% of the patches were merged. 

Someone from the audience spoke up to say that many of the network driver patches get merged without much review because there is no dedicated maintainer. That makes the latency lower, but things get merged with lots of bugs. Sang said that is one way to deal with a lack of reviewers: if the network driver patch looks "halfway reasonable" and no one complains, it will often just be merged. If anyone is unhappy with that state of affairs, they should volunteer to help, another audience member suggested. Sang agreed, and said that taking on the maintenance of a single driver is a good way to learn. 

His subsystem is somewhere in between `net/ethernet` and `mtd`. That is "not bad", he said, for someone doing the maintenance in their spare time. But for a vendor trying to get SoC support upstream, it may not be quick enough. 

The dream, he said, would be for all subsystems to be more like the Ethernet drivers without accepting junk patches. His belief is that over time the latency in most subsystems is getting worse. In fact, his "weather forecast" is that we will see more and more problems over time, either with increased latency or questionable patches going into the trees. 

So, what can be done to help out maintainers? For users, by which he means people who are building kernels for customers, not developers, necessarily, or regular users, he recommends giving more feedback to subsystem maintainers. Commenting on patches, testing them, and describing any problems found will help. Add a Tested-By tag if you have done that, as well. If there is no reaction to a patch on the mailing list and seemingly no interest, it makes his job of deciding whether the patch is worthwhile difficult. If you are using a patch that hasn't been merged, consider resending it, but check to see if there are open issues from when the patch was posted. Sometimes there are simple style changes needed that can be easily fixed. 

For developers, he recommends trying to get the patch right the first time and thus reducing the number of superseded patches. Not knowing the subsystem and making mistakes that way is reasonable, but sloppy patches are not. In addition, if you know a patch is a suboptimal solution to the problem, be honest about it. Don't try to sell him something that you know is bad. Sometimes a lesser solution is good enough, but a straight explanation should accompany it. 

He also recommends that developers take part in the patch QA process by reviewing other patches. In fact, he said, you should also review your own patches as if they came from someone else—it is surprising what can be found that way. Taking part in the mailing list discussions, especially those that are about the architecture of the subsystem, is important as well. It is difficult to determine which way to go, at times, without people stating their opinions. 

Maintainers should not necessarily work harder, Sang said, as most are working hard already. It is important to watch out for burnout as no one wins if that happens. SoC vendors are "constantly pressing the fast-forward button" by releasing hardware faster and faster, so you may reach a point where you simply can't keep up. That may be time to look for a co-maintainer. 

Having the right tools is an important part of being a maintainer. There is no "ready-made toolbox" that is handed out to new maintainers, but if you talk to other maintainers, they may have useful tools. Keyboard shortcuts, Git hooks for doing auto-testing, tools to handle and send out email, and so on are all time savers. "Pay attention to the boring and repetitive tasks" and try to automate them. 

Organizations like the Linux Foundation (LF), Linaro, SoC makers, and others have a role to play as well. If they already have developers, it is important to allow those developers to review patches and otherwise participate in kernel QA. That will improve the developers' skills, which will help the organization, and it will improve the kernel too. 

It is important to educate new kernel developers internally about the basics of kernel submissions. He is much more lenient with someone who he knows is working on his own than he is with those working at companies where there are multiple folks who already know about submitting patches and could have passed the knowledge on. 

Increasing the number of maintainers would help as well. It might be easier for people to take on maintainer responsibilities if it were part or all of their job to do so. Sang believes that being a maintainer should ideally be a full-time paid position, but that is often not the case. He does it on his own time, as do others, and some do it as part, but usually not all, of their job. A neutral party like the LF might be desirable as the employer of (more) maintainers, but other organizations or companies could also help out. In his mind, it is the single most important step that could be taken to improve the kernel maintainer situation. 

He went back to the SoC diagram he showed early on, but this time colored the different subsystems based on whether the maintainer was being paid to do that work. Red meant that the maintainer was doing it in their spare time, and there were quite a few subsystems in that state. That is somewhat risky for SoC vendors trying to get their code upstream. Ideally, most or all of the diagram would be green (maintainer paid to do it) or yellow (part of the maintenance time is paid for). Sang ended by saying that having full-time maintainers was really something whose time had come and he is optimistic that more of that will be happening soon. 

[I would like to thank the Linux Foundation for travel assistance to Edinburgh for the Embedded Linux Conference Europe.]
