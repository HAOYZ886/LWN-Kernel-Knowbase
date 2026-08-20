---
title: The Civil Infrastructure Platform after (nearly) ten years
url: https://lwn.net/Articles/1049604/
date: "December 17, 2025"
category: "Civil Infrastructure Platform; Long-term support initiative"
author: "By Jonathan Corbet December 17, 2025 OSS
Japan"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 17, 2025

* * *

[OSS Japan](<https://lwn.net/Archives/ConferenceByYear/#2025-Open_Source_Summit_Japan>)

The [Civil Infrastructure Platform](<https://cip-project.org/>) (CIP) first launched in that form in April 2016, so it has a tenth-anniversary celebration in its near future. At the 2025 [Open Source Summit Japan](<https://events.linuxfoundation.org/open-source-summit-japan/>), Yoshitake Kobayashi talked about the goals of this project and where it is headed in the future. Supporting a Linux system for even one year is a challenging task; maintaining that support for a decade or more is rather more so, and a changing regulatory environment complicates the task further. 

The mission of CIP is to provide "industrial-grade Linux" as an open-source base layer, Kobayashi began. CIP has run up a few achievements in its first ten years, starting with the "super long-term support" (SLTS) kernels, which are supported for a minimum of ten years. CIP has been working toward alignment with industry standards, and [IEC 62443](<https://www.isa.org/standards-and-publications/isa-standards/isa-iec-62443-series-of-standards>) (which is concerned with ""requirements and processes for implementing and maintaining electronically secure industrial automation and control systems"") in particular. The project has also made significant upstream contributions to projects like Debian and KernelCI. 

#### The longevity gap

[![\[Yoshitake
Kobayashi\]](https://static.lwn.net/images/conf/2025/ossjp/YoshitakeKobayashi-sm.png)](<https://lwn.net/Articles/1049605/>) Civilization, he said, runs on Linux. There are vast numbers of hidden industrial systems running Linux in many settings, including energy, transportation, building automation, and more. When CIP had its beginnings in the early 2010s, industries using Linux in this way were contending with a ""longevity gap""; kernel releases are frequent, but even the kernel's long-term support runs out after six years in the longest (and discontinued) case. Power plants, railways, and other systems with Linux inside can run for ten, 20, or even 50 years, though. That has led companies to create — and to have to maintain — their own proprietary Linux forks, with the usual costs and security risks. 

The CIP concept was first presented ([slides](<https://events.static.linuxfound.org/sites/events/files/slides/Applying%20Linux%20to%20the%20Civil%20Infrastructure-LinuxCon%20Japan%202015.pdf>)) at LinuxCon Japan in 2015. The requirements at that time included a minimum of ten years of support, ongoing security updates, and a kernel with realtime capabilities. Over time those requirements have evolved, but they remain focused on industrial-grade reliability, functional safety, and realtime response. To get there, CIP has created an open-source base layer, a sort of minimal Linux distribution. This layer includes the CIP kernel, and a small set of core packages. This base layer, when used by companies in their project, can bring about a 70% reduction in the effort required to create and maintain the resulting system, he said. 

CIP's history can be split into three phases, he said. The first, through 2017, was mostly focused on defining policies for the project. From 2018 to 2021, the effort went into the creation of working groups and the implementation of the base layer. Since 2022, the focus has been on compliance and resilience work. 

#### The pillars

The project's working groups comprise the pillars that hold the whole thing up. The first is the kernel effort which, he said again, seeks to project a minimum of ten years of support. That work necessarily involves backporting a lot of patches, but the project's policy requires that any backported patches must first land in the mainline kernel. While most backports are fixes of one type or another, there is also a certain amount of work done to support newer hardware in older kernels. 

The first SLTS kernel was 4.4, which was released in 2016; it was first adopted by CIP in 2017. Initially the support work was done by Ben Hutchings, but then it moved over to the CIP kernel team. This kernel will hit end of life in January 2027. It was intended to be a proof-of-concept showing that extended support of a kernel in an open setting can work; now, the project is supporting five SLTS kernels (the others are 4.19, 5.10, 6.1, and 6.12). For as long as those kernels have normal long-term support, CIP does not have much work to do; the project will take the kernels over once the regular support ends. 

The kernel team, he said, is currently reviewing over 1,000 patches per month for backport consideration. The team also looks at about 2,000 CVE entries per month, just for the 6.1 kernel. There have been 466 SLTS releases to date. There are 11 boards supported by the five-person team working on the SLTS kernel. As an example of how this support has worked, he put up a slide showing each 4.4 release, indicating how many patches were backported by the CIP project itself; those comprise all of the patches applied, of course, once the community support for 4.4 ended. 

> ![\[4.4 kernel patches\]](https://static.lwn.net/images/conf/2025/ossjp/cip-slide.png)

The security process involves reviewing huge numbers of CVE entries, many of which are not applicable to the CIP kernel. The first review step is automated; it takes the CIP kernel configuration into account to weed out the CVEs that cannot be applicable. The remainder must then be manually reviewed. 

The CIP Core Working Group is charged with providing the reference base image — the kernel with the core utilities on top of it. There was no desire within the project to create an entirely new distribution, so CIP chose to work with the Debian project instead. CIP's efforts help with Debian's long-term support, and continue after Debian moves on. There are two system profiles — "tiny" and "generic" — maintained by CIP, but the tiny profile is being phased out. As the capabilities of embedded systems have grown, the need for an extra-small base image has decreased. There are currently five Debian releases supported by this group. 

Kobayashi pointed out that the reference images are created with a reproducible build process; there is a strong desire to keep the process transparent and ensure the the result can be trusted. 

The Testing Working Group has put together a system called [Board At Desk (B@D)](<https://cip-project.org/blog/2017/06/01/board-at-desk-bd-and-forthcoming-challenges>), which allows developers to connect boards to the central continuous-integration (CI) system. Developers can use B@D to test changes on real hardware from their own desktops. The working group has been building a centralized testing infrastructure, using GitLab runners, that is integrated with the KernelCI project. The results from CIP testing can be seen on the KernelCI site. 

The Security Working Group is focused on the requirement that the CIP core image needs to be a _secure_ reference image. There is an emphasis on IEC 62443 compliance; the hope is that a compliant base image will be helpful to users seeking their own compliance certification. The project has also put together some guidance for its users to help them obtain that certification. The IEC 62443-4-1 assessment of the base was completed in August 2024; the IEC 62443-4-2 assessment is underway now, with a hoped-for completion in 2026. 

The Software Update Working Group is charged with the creation of a robust update framework for CIP-based systems. This work involves integrating with systems like [SWUpdate](<https://sbabic.github.io/swupdate/swupdate.html>), [TUF](<https://theupdateframework.io/>), and [wfx](<https://github.com/siemens/wfx?tab=readme-ov-file#wfx>). The project has implemented A/B updating (maintaining two independent images allowing fallback to a working system if an update fails) and delta updates. Integration of TUF is done now; wfx integration is in progress. 

#### The road ahead

Kobayashi concluded with a brief look forward — which actually started with the recent past. CIP first integrated the realtime preemption patches in 2017; the feature has been officially supported since late 2024, shortly after the [completion](<https://lwn.net/Articles/990985/>) of the realtime preemption merge. 

The near-term future of CIP, beyond maintaining all those images, appears to be dominated by the coming ""regulatory wave"". That wave takes the form of the European Cyber Resilience Act (CRA) in the near future. The CIP base system, he said, will serve as a sort of shelter for manufacturers, helping them to provide the updates mandated by the CRA. The plan, he said at the end, is to evolve CIP into a ""compliance base"" maintained as an open-source project. 

[The slides from this presentation](<https://static.sched.com/hosted_files/ossjapan2025/53/20251208-Towards%20a%20Decade%20of%20Industrial%20Grade%20Linux-OSSJ-Yoshi.pdf>) are available. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to this event.]
