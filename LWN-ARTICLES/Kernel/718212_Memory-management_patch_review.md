---
title: "Memory-management patch review"
url: https://lwn.net/Articles/718212/
date: "March 29, 2017"
category: "Development model-Code review"
author: "By Jonathan Corbet March 29, 2017 LSFMM 2017"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 29, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017>)

Memory-management (MM) patches are notoriously difficult to get merged into the mainline kernel. They are subjected to a high degree of review because this is an area where it is easy to get things wrong. Or, at least, that is how it used to be. The final memory-management session at the 2017 Linux Storage, Filesystem, and Memory-Management Summit was concerned with patch review in the MM subsystem — or the lack of it. 

Michal Hocko started the session off by saying that too many patches get into Andrew Morton's ‑mm tree without proper review. Fully half of them, he said, lack an `Acked-by` or `Reviewed-by` tag. But that is only part of the problem: even when patches _do_ carry tags indicating that review has been done, that review is often superficial at best, focusing on little details. Reviewers are not taking the time to think about the real problem, he said. As a result, MM developers are "building hacks on other hacks because nobody remembers why they were added in the first place". 

As an example, he raised memory hotplug, and the care that is taken when shifting pages between memory zones. But much of that could be avoided by simply not assigning pages to zones as early as happens now. MM developers were used to working around this issue, he said, and so never really looked into it. In the end, this is turning the MM subsystem into an unmaintainable mess that is getting worse over time. How, he asked, can we get more review for MM patches, as befits a core kernel subsystem? How can we get review that really matters, and how can we force submitters to fix the problems that are found? 

One option, Hocko said, is to make it mandatory that every MM patch have at least one review tag. That, he said, is likely to slow things down considerably. There are 100-150 MM patches merged in each development cycle; if the 50% or so of them without review tags are held back, a lot less will get merged. Is the community OK with that? 

Kirill Shutemov said that, if reviews are required to get patches merged, there will also need to be a way to get developers to do those reviews. Hocko agreed, saying that few developers are reviewing patches now. Mel [![\[Michal Hocko\]](https://static.lwn.net/images/conf/2017/lsfmm/MichalHocko2-sm.jpg)](<https://lwn.net/Articles/718214/>) Gorman said that requiring reviews might be fair, but there should be one exception: when developers modify their own code. In general, the principal author should not need as much review for subsequent modifications. 

Morton said that a lot of patches do not really require review; many of them are trivial in nature. When review does happen, he said, the quality can vary considerably; there are some `Reviewed-by` tags that he doesn't believe at all. Gorman agreed that reviews need to have some merit to be counted. 

Johannes Weiner worried that requiring reviews could cause work to fall through the cracks. Obscure bug fixes might not get merged, and memory-hotplug work could languish. Memory hotplug is a particular "problem child", Morton said; there is a lot of drive-by work and he has no idea who can review it. Quite a few people, Hocko added, are pursuing their own use case and don't really care about the rest. Part of the problem, Morton said, is that nobody really wants to clean up memory hotplug and, even if they did, they don't have the hardware platforms that would allow them to test the result. 

Gorman said that it is important to start enforcing some sort of rule around review. Patches that still need review should have a special tag in the -mm tree. If the percentage of patches so tagged is too high when the -rc5 prepatch comes out, developers who have pending patches should be conscripted to do some review work. That would, at least, encourage the active developers to do a bit more review work. 

Hocko then went back to the issue of trivial patches which, he said, are a bigger problem than many people think. Many of them are broken in obscure ways and create problems. Gorman suggested simply dropping trivial patches that have no user impact. Morton said that he could make an effort to be more careful when taking those patches, but that his attempts to get reviews for these patches are often ignored. If the people who have touched a certain file ignore a patch to it, Gorman said, that patch should just be dropped. 

Morton replied that he is reluctant to mandate a system where it's impossible to get changes into the kernel if you can't get them reviewed. People get busy or take vacations, and many of those patches are changes that we want anyway. Dropping them would be detrimental to the kernel as a whole. Hocko said that XFS is now mandating reviews for all changes, and it doesn't appear to be suffering from patches being dropped on the floor. 

The discussion then shifted to high-level design review, with Hocko saying that high-level review is hard and he wishes we had more of it, but it is not the biggest problem. The real issue is that we have more submitters of changes than reviewers of those changes. Morton said that he would push harder to get patches reviewed, and would do a walk-through around -rc5 to try to encourage review for specific patches needing it. 

Morton said there are particular problems around specific patch sets that never seem to get enough review. [Heterogeneous memory management](<https://lwn.net/Articles/717614/>) is one of those; it is massive, and somebody spent a lot of time on it, but there don't seem to be a whole lot of other people who care about it. The longstanding [`ZONE_CMA`](<https://lwn.net/Articles/684611/>) patches are another example. There is a demand for this work, but it has been blocked, he said, partly because Gorman doesn't like it. Gorman replied that he still thinks it's not a good idea, and "you're going to get a kicking from it", but if the people who want that feature want to maintain it, they should go for it; it doesn't affect others. So he will not block the merging of that code. 

Hocko raised the topic of the hugetlbfs code, which is complex to the point that few developers want to touch it. Perhaps, he said, hugetlbfs should be put into maintenance mode with no new features allowed. The consensus on this idea seemed to be that the MM developers should say "no more" to changes in this area, but not try to impose strict rules. 

Another conclusion came from Morton, who encouraged the MM developers to be more vocal on the mailing lists. The volume on the linux-mm list is reasonable, so there is no real excuse for not paying attention to what is happening there. Developers should, he said, "hit reply more often". Gorman agreed, but said that there need to be consequences from those replies; if a developer pushes back on a patch, that patch needs to be rethought. 

By that time, the end of LSFMM was in sight, and thoughts of beer began to take over. Whether this discussion leads to better review of MM patches remains to be seen, but it has, if nothing else, increased awareness of the problem.
