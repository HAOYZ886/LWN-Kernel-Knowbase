---
title: "Super long-term kernel support"
url: https://lwn.net/Articles/749530/
date: "March 19, 2018"
category: "Long-term support initiative"
author: "By Jonathan Corbet March 19, 2018 ELC"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 19, 2018

* * *

[ELC](<https://lwn.net/Archives/ConferenceByYear/#2018-Embedded_Linux_Conference>)

Some years ago, prominent community leaders [doubted](<https://lwn.net/Articles/126782/>) that even short-term stable maintenance of kernel releases was feasible. More recently, selecting an occasional kernel for a two-year maintenance cycle has become routine, and some kernels, such as 3.2 under the care of Ben Hutchings, have received constant maintenance for as much as six years. But even that sort of extended maintenance is not enough for some use cases, as Yoshitake Kobayashi explained in his Embedded Linux Conference talk. To meet those needs, the [Civil Infrastructure Platform (CIP)](<https://www.cip-project.org/>) project is setting out to maintain releases for a minimum of 20 years. 

CIP, he said, is one of the most conservative projects out there, but also one of the most important. It is working to create a stable base layer for civil infrastructure systems. It is not trying to create a new distribution. Civilization runs on Linux. Infrastructure we all count on, including that dealing with transportation, power generation, and more, is Linux based. If those systems fail, we will have serious problems. But this kind of infrastructure runs on a different time scale than a typical Linux distribution. The development time required just to place such a system in service can approach two decades, and the system itself can then stay in service for 25-60 years. 

The computing systems that support this infrastructure must thus continue to work for a long time. It must be based on "industrial-grade" software that is able to provide the required level of reliability, robustness, and [![\[Yoshitake
Kobayashi\]](https://static.lwn.net/images/conf/2018/elc/YoshitakeKobayashi-sm.jpg)](<https://lwn.net/Articles/749532/>) security. But the systems supporting civil infrastructure also must be brought up to current technology levels. Until now, the long-term support needed to keep them running has been done by individual companies, with little in the way of shared effort. That has kept these systems functional, but it is an expensive approach that tends to lag behind the current state of the technology. 

The way to do a better job, Kobayashi said, is to put together a collaborative framework that supports industrial-grade software while working with the upstream development communities as much as possible. That is the role that the CIP was created to fill. There are currently [seven member companies](<https://www.cip-project.org/members>) supporting CIP, with Moxa being the latest addition. They are supporting the project by contributing directly to upstream projects and funding work that advances CIP's objectives. 

CIP is currently focused on the creation of an open-source base layer consisting of a small number of components, including the kernel, the GNU C library, and BusyBox. Distributors will be able to build on this base as is needed, but CIP itself is starting small. The primary project at the moment is the creation of the super-long-term support (SLTS) kernel which, it is hoped, can be supported for at least ten years; as experience with extra long-term support grows, future kernels will have longer periods of support. The first SLTS kernel will be based on the 4.4 LTS release and will be maintained by Ben Hutchings; the [4.4.120-cip20](<https://lists.cip-project.org/pipermail/cip-dev/2018-March/000948.html>) release came out on March 9\. 

For the most part, the CIP SLTS kernel will be based on vanilla 4.4, but there are some additions being made. The latest Meltdown and Spectre fixes are being backported to this kernel for example, as are some of the hardening patches from the Kernel Self-Protection Project. Support for some Siemens industrial-control boards is being added. Perhaps the most interesting enhancement, however, is the realtime preemption patch set, which is of interest for a number of the use cases targeted by the CIP project. CIP has joined the realtime preemption project as a member and is planning to take over the maintenance of the 4.4-rt kernel. The first SLTS kernel with realtime support was released in January. 

In general, the project's policy will be to follow the upstream stable releases for as long as they are supported. Backports from newer kernels are explicitly allowed, but they must be in the mainline before being considered for addition to an SLTS kernel. New kernel versions will be released every four-to-six weeks. There is an explicit policy of non-support for out-of-tree drivers; distributors and users can add them, of course, but any bugs must be demonstrated in a pristine SLTS kernel before the CIP project will act on them. 

A new major kernel release will be chosen for super-long-term support every two or three years. The project is currently thinking about which release will be the base for the next SLTS kernel; for obvious reasons, alignment with upstream LTS choices is important. There will be a meeting at the Japan Open Source Summit to make this decision. 

There is some initial work on testing infrastructure based on the "board at desk" model; the testing framework is based on the [kernelci.org](<https://kernelci.org/>) infrastructure. Future work includes collaboration with other testing efforts, more frequent test coverage, and support for container deployment on SLTS-based systems. Debian has been chosen as the primary reference distribution for CIP systems, and all of the CIP core packages have been taken from Debian. As part of this effort, CIP is supporting the [Debian-LTS effort](<https://wiki.debian.org/LTS>) at the platinum level. 

The CIP core effort is working on the creation of installable images consisting of a small subset of Debian packages and the CIP SLTS kernel. This work can be found [on GitLab](<https://gitlab.com/cip-project/cip-core>). CIP is working with Debian to provide longer-term support of a subset of packages, to improve cross-compilation support, and to improve the sharing of [DEP-5 license information](<https://wiki.debian.org/Proposals/CopyrightFormat>). 

In the longer-term, CIP is looking toward [IEC-62443](<https://en.wikipedia.org/wiki/Cyber_security_standards#ISA/IEC-62443_\(formerly_ISA-99\)>) security certification. That is an ambitious goal and CIP can't get there by itself, but the project is working on documentation, test cases, and tools that will hopefully help with an eventual certification effort. Another issue that must be on the radar of any project like this is the [year-2038 problem](<https://lwn.net/Articles/717076/>), which currently puts a hard limit on how long a Linux system can be supported. CIP is working with kernel and libc developers to push solutions forward in this area. 

Someday CIP hopes to work more on functional safety issues and to come up with better solutions for long-term software updates. The project has just joined the [EdgeX Foundry](<https://www.edgexfoundry.org/>) to explore what common ground may be found with that group. Clearly, the CIP project has a lot of issues on its plate; it seems likely that we will be hearing about this project for a long time. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting your editor's travel to ELC.]
