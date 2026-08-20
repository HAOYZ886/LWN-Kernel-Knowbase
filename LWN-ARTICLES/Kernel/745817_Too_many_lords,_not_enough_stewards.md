---
title: "Too many lords, not enough stewards"
url: https://lwn.net/Articles/745817/
date: "January 31, 2018"
category: Development model
author: "By Jake Edge January 31, 2018 linux.conf.au"
---

By **Jake Edge**  
January 31, 2018

* * *

[linux.conf.au](<https://lwn.net/Archives/ConferenceByYear/#2018-linux.conf.au>)

For anyone who has followed Daniel Vetter's talks over the last year or two, it is fairly clear that he is not happy with the kernel development process and the role played by kernel maintainers. In a strongly worded talk at linux.conf.au (LCA) 2018 in Sydney, he further explored the topic (that he also [raised at LCA 2017](<http://blog.ffwll.ch/2017/01/maintainers-dont-scale.html>)) in a talk entitled "Burning down the castle". In his view, kernel development is broken and it is unlikely to improve anytime soon. 

He started by noting that this talk would be a "rather more personal talk than others I give". It is his journey from first looking in on the kernel in high school to learn how operating systems work. The kernel developers were his heroes who created this awesome operating system by discussing things out in the open. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Eventually he started scratching his own itch in the graphics subsystem, which led to him getting hired to work on Linux graphics professionally on a small team. He got volunteered to be the kernel maintainer for that team, which grew from three to twenty people in a year or two. In that time he learned the tough lesson that "leading teams is leading people". But he has learned that the way kernel maintainers work is making developers unhappy, including him. The talk would be a look at how he learned just how broken things are. 

#### What's broken

The first thing that generally comes up when discussing what's broken in the kernel community is the discussion culture, for example Linus Torvalds cursing at someone. That culture is enshrined in the kernel's [Code of Conflict](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b0bc65729070b9cbdbb53ff042984a3c545a0e34>), which says that if you want to contribute to the Linux kernel, "you will get shredded for the greater good of the project", he said. There is another paragraph that says if you get really uncomfortable, you can report it to the Technical Advisory Board (TAB) "and they might not be able to do a whole lot about it", he said. 

[ ![\[Daniel Vetter\]](https://static.lwn.net/images/conf/2018/lca/DanielVetter-sm.jpg) ](<https://lwn.net/Articles/745936/>)

But if you talk to kernel developers at conferences, they will often say that the culture has gotten much better in the past few years. Generally what they mean, Vetter said, is the "rather violent language and discussion" in the kernel community has decreased or disappeared. But he thinks that is not the real problem, it is only one aspect of the problem. 

That led to his first interlude, which was about a book that he read as part of his thinking about Linux kernel culture. It was [_Why Does He Do That?: Inside the Minds of Angry and Controlling Men_](<https://www.goodreads.com/book/show/224552.Why_Does_He_Do_That_>) by Lundy Bancroft. The interesting part of that book is the archetypes of abusers that Bancroft has extracted and the patterns of behavior that abusers engage in so that they can stay in power, he said. Vetter's main takeaway from the book is that abuse comes down to two elements. 

First, the abuser must have power over the victim; in the kernel case, maintainers have a lot of power over their contributors, since their right to reject code is absolute. To get code into the upstream kernel, developers have to deal with the maintainer of the subsystem. 

The other element is controlling behavior, which is not necessarily violence. Clearly, violence can be completely controlling behavior that puts the safety of the victim at risk, but there are counterexamples such as martial arts. Those are violent but, if done correctly, respectful. There is plenty of non-violent controlling behavior, though, including determining who a victim can talk to or go out with. 

Vetter wanted to highlight the kinds of controlling behavior he has seen, which maintainers use to dictate what their contributors can and cannot do. That list starts with the assertion that "only technical topics are in-scope" for the kernel community. This is nonsense, he said; for developers it is true, but it may not be for maintainers. 

For some maintainers, that means anger, screaming, and shouting, but others expect emotional support because they are overloaded. The emotional state of the developers is totally irrelevant. There is also a lot of micro-aggression, nagging, and bikeshedding from maintainers in an environment that is ostensibly technical-only. Maintainers can impose their emotional state on their contributors in the mailing list, but only the maintainers' emotions matter, not contributors'. This is classic controlling behavior, Vetter said. 

Beyond that, discussions about governance and fixing these problems are off-topic as well. That makes it hard to even discuss the problems with others. 

Leading teams of people is not a valued contribution within the kernel community, which creates a negative space for leadership, he said. Since maintainers are personally responsible for all tests failing, regressions in the subsystem, and so on, that makes things personal; it also turns maintainership into a high-stakes game. So maintainers self-censor and impose that on their sub-maintainers and contributors. Making it personal is, he thinks, a strong force that perpetuates the cycle of abusive and controlling behavior in the kernel community. 

This leads to something of a personality cult in some subsystems. There are people who have been working in the subsystem for many years and are the keepers of much of the knowledge and history built up over those years. These people are "very hard to remove". Pretty much every subsystem has their "local toxic person" that cannot be removed because of their accumulated store of knowledge. 

Maintainer power is not shared in most subsystems. The [group maintainership model](<https://lwn.net/Articles/705228/>) was pushed at Kernel Summits, but it has only been adopted in a few places: the x86 subsystem, ARM at the top level, and half of the graphics subsystem. In addition, maintainers are not documenting their implicit assumptions and rules. For example, even after writing, reviewing, and applying thousands of patches over the years, Vetter is still not sure how to format patches for other subsystems. Inevitably, there will be minor formatting or other issues that will arise when he submits patches elsewhere, but those rules are "outright not documented". 

The tools and tests that maintainers use to check patches are not made available, at least easily, for contributors to use. That is slowly getting better, he said. But it would be easier for developers to find and fix problems before they get to the maintainer's tree if the checking and testing tools were accessible. 

Most of the review that is done is between the maintainers and contributors. He crunched some numbers around a year ago and found that only 25% of review is done by peers, the rest is done by the maintainers. It becomes something of an exercise in conflict avoidance for the contributor, since the patch is on its way to acceptance. The contributor just needs to "go through the motions, respin the patch series ten times" to show that they are "sufficiently subordinate to the maintainer", he said, then it will be merged. 

For some suggestions on how to do things better, he recommended two talks. One is "Life is better with Rust's community automation" [[YouTube video](<https://www.youtube.com/watch?v=dIageYT0Vgg>)], given by Emily Dunham at LCA 2016. The other is "Have it your way: maximizing drive-thru contributions" [[YouTube video](<https://www.youtube.com/watch?v=q3ie1duhpCg>)] by VM Brasseur. The latter targets one-off contributions, but by making the process easier for one-time contributors, it will also improve things for regular contributors. His takeaway is that submissions should be looked at by a bot that points out any stylistic issues, perhaps even provides a fixed-up version of the code, and points contributors to the appropriate part of the documentation. So these rules are not only documented but also easily referred to when problems arise. 

#### Sinking feeling

The subsystem he was maintaining started growing wildly and he started to "get this sinking feeling that this is really not how you run a team". Around the same time, he was invited to his first Kernel Summit. He wanted to try to talk to some potential allies but, since graphics is kind of separate from the rest of the kernel, he did not know more than two or three of the hundred or so maintainers at the summit. He sought out some of those who had made public statements that made him think they might be up for "fighting the good fight" to change how kernel development is done. 

But he found it hard to enlist aid and allies at the summit. Some of the seemingly "nice maintainers" turned out to be resistant to change. There is a risk in speaking out in the kernel community, he said. So it turns out that there are few maintainers willing to do so. When you chat with many of the people who have been around kernel development for a long time and challenge them about problematic people, toxic behavior, or "maybe we should do this better" the response is often "it is what it is and you just deal with it". In his view, this makes them complicit. 

The people that you can talk to, Vetter said, are those who have left the community. That is great because you can compare notes. When you do so, the same names often come up as maintainers that are something of a disappointment for not doing more to help fix the problems. There are a lot of "loud quitters", but an even larger number of contributors have just dropped out silently. This provides a sizable group of people to talk to. 

Similarly, in some ways, is the pool of burnt-out maintainers. They realize the problems, but don't want to leave the kernel entirely so they step down as maintainers. You can have "really good conversations" with them, he said, but since they have already burned out, they do not have the energy to help out fighting the next fight. 

The participants on the dri-devel mailing list and the graphics subsystem in general have tried to do things a little differently to "make our little corner of the kernel more useful". The subsystem spearheaded the [rewrite of the kernel documentation system](<https://lwn.net/Articles/692704/>) and it is the only one that hands out commit rights. The latter makes graphics "more like a standard project". 

In the last year, a [code of conduct](<https://www.freedesktop.org/wiki/CodeOfConduct/>) has been enacted and is being enforced, he said. The "surprising thing is that it seems to work". There has been a need to quiet a number of kernel developers—he thinks they may end up banning some. If you make it clear that you are serious, some unreasonable people become much more reasonable, Vetter said. 

One group that would seem to have a strong incentive to fix the kernel process would be the "sandwiched maintainers". "They get the harassment from above and they get the unhappy contributors from below." He has talked with other subsystems that would like to see things be done differently, but it seems that once people start talking about fixing things it just eventually peters out and "nothing happens". 

Before getting into the forces that tend to cement kernel development place, Vetter provided a few places to learn about maintainership. The [Community Leadership Summit](<http://www.communityleadershipsummit.com/>) and the affiliated CLSx events, which are held at OSCON, LCA, and elsewhere, as well as the [Maintainerati](<https://maintainerati.org/>) event, make for great places to talk with other open-source maintainers. The community tracks at many different conferences are also good. One author he wanted to highlight is [John Kotter](<https://www.kotterinc.com/team/john-kotter/>), whose books on change management have been helpful to Vetter's thinking on how to change an existing community with a lot of inertia. 

#### Strong forces

There are some strong forces that uphold the status quo in the kernel, he said. One is the spectator sport nature of the kernel mailing list. When Torvalds blows up publicly, it is picked up by Reddit, Hacker News, and elsewhere. It is something of an "abusive performance art" that reinforces the personality cults and shows the power that maintainers wield. It also demonstrates the high-stakes nature of speaking out, since every time one of Torvalds's rants gets posted, all of the previous episodes from years ago are rehashed. 

Some maintainers are so scared of the next blow up, and have the scars from the last one, that they are unwilling to change anything because that's the safest option. So if the subsystems starts talking about group maintainership or handing out commit rights, it eventually boils down to the maintainer saying "no". They are worried that if a regression happens or there is some other problem that occurs due to the change, they will be lambasted publicly (again). They are trying to handle their fear by keeping as much control as possible, which just perpetuates things, he said. 

A lot of maintainers are employed because they are the bottleneck. Their managers should be forcing them to stop being the bottleneck by adopting commit rights and simply being the one who gets the blame when Torvalds freaks out, but that is not happening. The maintainer bottlenecks continue to have job security and the managers seem to be satisfied with that approach. 

The Linux Foundation (LF) is part of the problem as well, he said. It was set up, partly, because some were concerned about the amount of power Torvalds was wielding, so they wanted a neutral foundation that would employ him. The steep hierarchy in the kernel-development world is an advantage for the foundation, because it can employ the top maintainers and thus provide exclusive access to them for its members and others. More recently, the LF has moved into the cloud world, so this is less of an issue, but the LF is still a factor in maintaining the hierarchy, he said. 

As the secretary for the X.Org Foundation, he has heard some of the old stories where that foundation ran into difficulties along the way, so he sees conflicts of interest as inevitable. If the project governance and business sides are intertwined, things can get messy; after ten years, the community may have moved on, but the business is still stuck in the older thinking. So instead of best serving the community, the foundation's goal becomes one of keeping the people employed, even though there may not be a need for those jobs anymore. 

In Vetter's opinion, there should be a strict separation between project governance and employing people. So the foundation would provide services and infrastructure for the project but not employ people to work on the project directly. Because, at some point, those people may not be doing work that is in the overall interest of the project and its community any longer. Similarly, he thinks that voting rights should be spread widely, so that when there are shifts in the community, the new people are not ignored because they have no voting power. 

He wrapped up his talk by noting that the maintainers benefit from the status quo, so they are unlikely to try to change it. He would not suggest that contributors not work on the kernel, as it is a massive career boost, but did suggest they always have an exit plan. "If you stick out your head, and sooner or later you will stick out your head, it will get chopped at." 

#### Steward, not lord

His talk title could be taken different ways; it could be seen as a reference to burn it all down and start over. But he sees things differently; it is a plea to the maintainers to please fix things before the revolution comes. That revolution would be a terrible thing for Linux. The current leadership refuses to see the problems, he said, and gets defensive when they get brought up. 

So he suggested that maintainers share power, drop privileges, and don't make their reviewer powers special. They should also document everything they can and automate things, as well. It is important to "care about the people", because without people you don't have much of a project. He summed up his suggestions with "be a steward, not a lord". 

He then answered a few audience questions. It is unclear to him how and why things have come to be this way. It may be that long ago, when Linux was started, we didn't know how to build a working community, as we do today. By the time people started speaking up, the power structure was too entrenched to be changed. 

Vetter said he has a hard time recommending that people join or rejoin the kernel community. Outside of the graphics subsystem, the overall understanding of the problems is so lacking that you can't even start talking about solutions. One step might be to get people to a point where they agree that just apologizing for the current situation is not sufficient. Maybe there is another subsystem that could start to make positive changes; if that happens, it would make his talk a "resounding success", he said, but the time frame for that is probably something like five years. 

There are probably elements of the talk that almost any participant or observer would quibble (at least) with, but it seems fairly likely that there would not be widespread agreement on which. The reaction to the talk has been laudatory in many quarters and he clearly voiced concerns that have largely stayed under wraps. In the talk, Vetter has identified some real problems in the kernel community, it remains to be seen where it goes from here. 

A [YouTube video](<https://www.youtube.com/watch?v=BB0luXmuo3g>) of the talk is available. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to Sydney for LCA.]
