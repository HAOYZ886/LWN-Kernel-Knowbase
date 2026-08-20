---
title: Group maintainership models
url: https://lwn.net/Articles/705228/
date: "November 2, 2016"
category: "Development model-Maintainers"
author: "By Jonathan Corbet November 2, 2016 2016 Kernel Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 2, 2016

* * *

[2016 Kernel Summit](<https://lwn.net/Articles/KernelSummit2016/>)

Traditionally, kernel subsystem maintainership is a solitary job, but there has been a steady increase in the number of subsystems that are using some sort of group model instead. At the 2016 Kernel Summit, Darren Hart and Daniel Vetter talked about how these models work in practice and what their experiences might have to offer other subsystem maintainers. 

Darren started by noting that there are a number of motivations behind group maintainership, starting with the fact that the work, for a busy subsystem, can often be more than one person can handle. Some sort of load balancing can help to keep maintainers from burning out. Group models are [![\[Darren
Hart\]](https://static.lwn.net/images/conf/2016/ks/DarrenHart-sm.jpg)](<https://lwn.net/Articles/705229/>) also more robust in the face of vacations, illness, or simply a day job that gets busy. Dan Williams added that group maintainership can also be a good way to develop new maintainers for the future. 

There are, Darren said, two models of group maintainership seen in the kernel community. One of them is the "hands off" model, as exemplified by the arm-soc tree maintained by Arnd Bergmann and Olof Johansson. They manage a single repository, using an IRC channel to take a "lock" when they are ready to apply some changes. They maintain a log file, Olof said, so that they can always see what the other has done. 

The other model is "delegation," usually seen in subsystems that use the [patchwork](<http://jk.ozlabs.org/projects/patchwork/>) patch-management subsystem. Patchwork can delegate the handling of each patch to a specific maintainer; Darren would like to start making more use of it. Mauro Carvalho Chehab said that this is the approach used in the media subsystem; there are two maintainers, and patches are automatically delegated by patchwork. Rafael Wysocki added that the power-management subsystem also uses it; in this case, the power-management mailing list is shared between multiple subsystems, so the automatic delegation in patchwork helps to sort out changes as they arrive. 

Daniel Vetter talked a bit about the multiple-committer model used in the i915 graphics driver subsystem; it was a shortened version of the talk that was covered in [this article](<https://lwn.net/Articles/703005/>) in October. He had been working in a two-person team (with Jani Nikula) for three years, but wanted more help. He had plenty of reviewers, but couldn't find anybody else willing to be named as a co-maintainer. Patch submitters wanted to deal with the maintainer rather than with other reviewers, so he and Jani were becoming a bottleneck in the process. 

In response, they decided to try out a group model where many committers have the ability to commit changes to the repository. It is generally working well, though there has been "some fallout." The way that the tree is managed, with fixes being cherry-picked into another tree, creates trouble with linux-next; they have some ideas for how to improve that interaction. [![\[Daniel
Vetter\]](https://static.lwn.net/images/conf/2016/ks/DanielVetter-sm.jpg)](<https://lwn.net/Articles/705231/>) Developers are also occasionally confused when a seemingly random person accepts their patches. 

James Bottomley asked what the essential difference is between a committer and a maintainer in this model; Daniel answered that committers work internally, while the maintainer deals with the rest of the world. Committers in general don't want a lot of external visibility — they don't want to be listed in the MAINTAINERS file — so the solution is to call them something other than "maintainers." Ben Herrenschmidt observed that the maintainer's real job, in this model, is to accept the blame when things go wrong. 

Olof asked if Daniel had observed problems with developers shopping patches around trying to find an accommodating committer. Daniel responded that, in general, he trusts his committers to say "no." There had been a couple of cases involving managers who have tried to get patches merged that way; it seems to happen once with every new manager. His response is to set up a meeting with that manager and explain how things need to be done. When asked if arm-soc had that problem, Olof responded that their model, where they deal with submaintainers rather than taking patches from developers directly, tends to keep that from happening. 

The final part of the discussion centered on the workflow issues in the i915 subsystem that can cause Git to send patches multiple times — the core of the difficulty with linux-next. Daniel said that the tooling is not up to the job, but Linus responded that the workflow the group was using sounded "really nasty." What i915 is using, he said, is the submaintainer model; he should be taking pull requests from those maintainers rather than sharing a repository with them. Daniel said he is not against the submaintainer model, but it would create some coordination issues in this case; the nature of that driver (and DRM drivers in general) has a lot of developers working on the same files simultaneously. 

Linus insisted, though, that, with the right habits, the submaintainer model works. Maintainers should make use of topic branches and avoid back merges with upstream trees. Daniel agreed that the i915 model would not work well for proper subsystems, but for a "leaf" like the i915 driver, it works well. The session wound down at that point.
