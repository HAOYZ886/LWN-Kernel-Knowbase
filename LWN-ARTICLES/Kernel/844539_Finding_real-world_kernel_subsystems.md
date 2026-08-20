---
title: "Finding real-world kernel subsystems"
url: https://lwn.net/Articles/844539/
date: "February 1, 2021"
category: "Development model-Maintainers"
author: "By Jonathan Corbet February 1, 2021 LCA"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 1, 2021

* * *

[LCA](<https://lwn.net/Archives/ConferenceByYear/#2021-linux.conf.au>)

The kernel development community talks often about subsystems and subsystem maintainers, but it is less than entirely clear about what a "subsystem" is in the first place. People wanting to understand how kernel development works could benefit from a clearer idea of what actually comprises a subsystem within the kernel. In an attempt to better understand how kernel development works, Pia Eichinger (and her colleagues Ralf Ramsauer, Stefanie Scherzinger, and Wolfgang Maurer) spent a lot of time looking for the actual boundaries; Eichinger presented that work at the 2021 linux.conf.au online gathering. 

This work was undertaken to develop a more formalized model of how kernel development works. With such an understanding, it is hoped, ways can be found to make the process work better and to provide new tools. The researchers have a particular interest in safety-critical deployments of Linux. Safety-critical environments are highly sensitive; working software can make a life-or-death difference there. So safety-critical developers have to ensure software quality by any means available. 

One such means is to take a close look at the development process, on the reasonable assumption that the process impacts the quality of the final result. Assuming that the process itself makes sense, a project that adheres more closely to its defined process should [![\[Pia Eichinger\]](https://static.lwn.net/images/conf/2021/lca/PiaEichinger-sm.png)](<https://lwn.net/Articles/844547/>) produce higher-quality software. So if it can be proved that a project's developers strictly comply with their development process, the level of assurance is higher and certification — generally necessary for safety-critical systems — is easier to achieve. 

The Linux kernel presents some major challenges when it comes to certification due to its open development process. Nobody documents the process or the degree to which it is adhered to. But, she said, with a bit of data mining, much of that information can be recovered after the fact. Her focus is on patch integration in particular and whether patches are being merged by the appropriate subsystem maintainers. If patches are taking "strange paths", that is a sign that the process is not being followed. 

Eichinger ran into a little problem on the path to that goal, though: where can one find the subsystem hierarchy that defines this process? Where are the documents describing these subsystems; more to the point, what _is_ a subsystem, exactly? It may seem like a trivial question, she said; that is what the `MAINTAINERS` file is for. But it is not that easy; as was covered in [this article](<https://lwn.net/Articles/842415/>) (which she cited during the talk), the information in this file is neither complete nor 100% accurate. 

First of all, many kernel subsystems do not appear in `MAINTAINERS` at all. But the picture is less than clear even for those that are present. Consider, for example, the "media subsystem"; there is no entry for it. There are, however, over 100 `MAINTAINERS` entries with "media" in the name somewhere. Which of those is the true media subsystem? The answer is not clear for somebody who is not closely familiar with the kernel community. 

Eichinger and company needed a definition of a "subsystem", so they made their own. Entries in the `MAINTAINERS` file do not clearly describe subsystems, so they were deemed instead to be "sections" that describe some part of the kernel code base. Many of these sections share files with each other; those were designated as "thematically related". By finding and grouping clusters of related sections, the kernel's true subsystems could be found. 

To do so, she processed all of the section entries and plotted them on an undirected graph, where the sections themselves were the vertices and shared lines of code make up the edges. The initial graph looked like this (from [Eichinger's slides [PDF]](</images/conf/2021/lca/eichinger-lca.pdf>)): 

> ![\[Subsystem graph\]](https://static.lwn.net/images/conf/2021/lca/full-graph.png)

That was, she allowed, a bit messy. To try to create something more useful, she cut the graph down to the largest 20% of the sections in the `MAINTAINERS` file. The result for the aforementioned media subsystem looked like this: 

> ![\[media subsystem
graph\]](https://static.lwn.net/images/conf/2021/lca/media-subsystem.png)

Therein one sees a number of sections for specific drivers, including a sizeable sub-cluster in the staging directory and a small blob in the Android drivers. The section that ties it all together is "media input infrastructure" — the actual media subsystem. 

The picture for the direct rendering (DRM) subsystem looks a little different: 

> ![\[DRM subsystem
graph\]](https://static.lwn.net/images/conf/2021/lca/drm-subsystem.png)

This subsystem appears as a large collection of related small clusters, with a lot of overlap between them. She described this organization as "non-conforming" with the hierarchical subsystem model; it seems likely that what is actually seen here is the distributed, group-maintainer model used by the DRM developers. 

At this point, she has some sort of definition of subsystems, twelve of which were identified at the top level. Those twelve were the Arm architecture, drivers, crypto, USB, DRM, networking, media, documentation, sound, SCSI, more Arm stuff (OMAP architecture code, for example), and Infiniband. Along with that, she has a tool that can automate this sort of subsystem detection. It is, she said, "just scratching the surface" of the problem, but it is a start. 

There are a number of ways this work could go in the future. One would be to examine historical kernel releases to build a history of how kernel subsystems have evolved over time. This model can also be used, of course, for the original purpose of determining how well the actual kernel patch flow conforms to the maintainer model. There may be scope for applying this technique to other projects as well. 

For more information, readers can go to [Eichinger's bachelor thesis](<https://drive.google.com/file/d/12ta2YxgEzEfrIcmWid8kwIyVEywbUjbA/view>) describing the entire project. The code for performing this analysis (called "PaStA") can be found in [this GitHub repository](<https://github.com/lfd>).
