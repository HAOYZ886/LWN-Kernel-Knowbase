---
title: "A new era for memory-management maintainership"
url: https://lwn.net/Articles/1070994/
date: "May 7, 2026"
category: Development model
author: "By Jonathan Corbet May 7, 2026 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 7, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

On April 21, Andrew Morton [let it be known](<https://lwn.net/ml/all/20260421094216.8dfe14a8c62f2420fa5aace1@linux-foundation.org>) that he intends to begin stepping away from the maintainership of kernel's memory-management subsystem — a responsibility he has carried since before memory management was even seen as its own subsystem. At the 2026 Linux Storage, Filesystem, Memory Management, and BPF Summit, one of the first sessions in the memory-management track was devoted to how the maintainership would be managed going forward. There are a lot of questions still to be answered. 

Morton started by observing that he had received almost no responses to his announcement. That was, he suggested, a result of the fact that he was asking others to take on more responsibility in the subsystem; developers are going to have to pick up a lot of tasks. 

[![\[Andrew Morton\]](https://static.lwn.net/images/conf/2026/lsfmm/AndrewMorton-sm.png)](<https://lwn.net/Articles/1071173/>) How that work can be spread out is an open question. There are 164 C files in the kernel's `mm` directory; a quick search shows that terms like "THP" (transparent huge page), "cgroup" (control group), and "NUMA" each show up in a large number of those files. The core memory-management concepts, in other words, are widely spread throughout the subsystem. Moving code around might help a bit, Morton said, but not much; everything is heavily interlinked, so trying to split up the subsystem will be challenging. But, he noted with a smile, ""it's not my problem"". 

Regardless of how successful a split is, there will always be a need for a catch-all tree serving the subsystem as a whole. The management of that catch-all and integration tree will be picked up by David Hildenbrand; Morton offered his thanks to Hildenbrand for taking on that responsibility. 

The memory-management developers (and those for the kernel as a whole), he said, have many layers of defense to prevent the shipping of bad code to users. The community can put out random stuff, but it goes through weeks of testing in the `-mm` tree, followed by more weeks of testing after landing in the mainline kernel. Fixes can be backported into the stable kernels for years, and distributors provide further layers of assistance. More recently, [Sashiko](<https://lwn.net/Articles/1064830/>), by providing a new level of patch review, has become yet another layer of defense. 

Developers should, he said, recognize that they are not creating production-quality code at the outset; they are creating a technology and letting others turn it into products. All of the layers of defense allow developers to pursue an ""aggressive rate of change"". There is a downside to that change rate, though: it puts a lot of pressure on reviewers. Within the memory-management subsystem, the review work is quite lopsided, in that a small number of people do the bulk of the review, while many developers do not carry much of the review load at all. He does not understand why things work that way, and wishes that the situation could be improved. 

The memory-management team, he said, is a great group of people; they are cooperative and quick to help each other. He worries, though, that the community could, over time regress to be like other parts of the kernel or other open-source projects, where emails are ignored because maintainers are too busy. Ignoring messages from contributors, he said, is shameful and unacceptable. But, he thinks, the memory-management community does value its culture and will work to maintain it. 

Matthew Wilcox said that, when somebody asks him how to get started within the community, he always encourages them to start reviewing patches. That advice is often not taken, though. He added that, while he tries to respond to email, there are days where he is simply unable to reply to all of the messages that have shown up. Dan Williams said that the community has long depended on Morton to apply pressure when responses are needed; Hildenbrand and others will need to apply that pressure in the future. 

Hildenbrand responded that there will always need to be somebody who makes sure that people are doing their part; that's part of what makes the subsystem great. The level of developer frustration within memory management is lower than in many other subsystems. He worries a bit about the onset of LLM-based review tools, though. Everybody agrees that there is a need for more human reviewers, so he thinks that reviews from tools like Sashiko should not be posted to the public lists. Early-stage reviewers, who are just learning their way around, will be demotivated by seeing that automated reviews have already found many of the problems. Automated review, Hildenbrand said, should be one of the last lines of defense, not the first. 

[![\[David Hildenbrand\]](https://static.lwn.net/images/conf/2026/lsfmm/DavidHildenbrand-sm.png)](<https://lwn.net/Articles/1071174/>) Liam Howlett said that sending LLM-based reviews to one-off contributors can have the effect of validating bad ideas; a number of participants suggested that these tools are good at finding bugs, but less good at addressing the question of whether a given change should be made at all. Morton said that beginning reviewers often focus on understandability issues that more experienced developers will skip over; that, too, is important feedback. 

At the end of the session, Hildenbrand stepped up to thank the community for trusting him with the responsibility for running the integration tree. He warned Morton that the community was not going to let him go easily, though, there will be a lot of questions. Some sort of working group will be formed, Hildenbrand said, to figure out how the memory-management community's development model should work in the future. It may be that more sub-components will move to their own trees, while there will always be the integration tree to pull everything all together. He closed by letting it be known that he would not be doing all of the work on his own.
