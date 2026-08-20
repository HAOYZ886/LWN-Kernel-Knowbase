---
title: "The stable-kernel process"
url: https://lwn.net/Articles/799166/
date: "September 16, 2019"
category: "Development model-Stable tree"
author: "By Jonathan Corbet September 16, 2019 Maintainers Summit"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 16, 2019

* * *

[Maintainers Summit](<https://lwn.net/Articles/799060>)

The stable kernel process is a perennial topic of discussion at gatherings of kernel developers; the 2019 Linux Kernel Maintainers Summit was no exception. Sasha Levin ran a session there where developers could talk about the problems they have with stable kernels and ponder solutions. 

Levin begin by saying that he has been working on the complaints he got [the year before](<https://lwn.net/Articles/769253/>). One of those was that the automatic patch-selection system "goes nuts" and picks the wrong things. It has been retrained twice in the last year and has gotten better at only selecting fixes. About 50% of recent stable releases has been made up of patches explicitly tagged for stable updates; the other half has come from the automated system. 

One ongoing problem, he said, is that a lot of patches tagged for stable are not being backported properly. If a simple backport effort fails, Greg Kroah-Hartman sends an email to the people involved, who then have an opportunity to do the backport. But, by the time that happens, developers have moved on and are often unwilling to revisit that old work. Peter [![\[Sasha Levin\]](https://static.lwn.net/images/conf/2019/lpc/SashaLevin-sm.jpg)](<https://lwn.net/Articles/799167/>) Zijlstra said that he tends to ignore email about backport failures; he's not sure what else he should do with them. The answer, Levin said, is to send a working backport. 

Dave Miller said that he does all the backports himself for the last two stable releases. But then people come back asking for backports to old kernels like 4.4. He just doesn't have the time to try to backport changes that far. As a result, a lot of poor work gets into those older kernels. Thomas Gleixner said that he had to give up on backporting many of the Spectre fixes to the 4.9 kernel. Even some of the more recent fixes for speculative-execution problems are nearly impossible to backport despite being much cleaner code. Kroah-Hartman said that there are people who are paid to do that sort of work; it's not something that kernel developers should have to worry about. 

Levin said that he is trying to improve the backport process in general. He now gets alerts for patches that fix other patches that have been shipped in a stable update; those are earmarked for fast processing. He is also putting together a "stable-next" tree containing patches from linux-next that have been tagged for stable. It is intended to be an early-warning system for changes that will be headed toward the stable kernels in the near future. 

Jan Kara complained that he recently applied a fix to the mainline that had a high possibility of creating user-space regressions. He had explicitly marked it as _not_ being suitable for the stable updates, but it was included anyway. Levin replied that it is easy to miss those notes, along with other types of information like prerequisite patches for a given fix. There needs to be a better structure for that kind of information; he will be proposing some sort of tag to encapsulate it. 

That said, Levin made it clear that he would rather include even the patches that have been explicitly marked as being unsuitable for stable updates. If there are bugs in those patches, users will encounter them anyway once they upgrade. Holding the scarier patches in this way just trains users to fear version upgrades, which is counter to what the community would like to see. 

Ted Ts'o asked about the test coverage for stable releases; Kroah-Hartman answered that is is probably more comprehensive than the testing that is applied to the mainline. There are a lot of companies running tests on stable release candidates and reporting any problems they find. This testing goes well beyond basic build-and-boot tests, he said. 

The final topic covered was running subsystem tests on backports. The BPF subsystem, for example, has a lot of tests that are known not to work on older kernels, so nobody should be trying to do that. But fixes to tests are backported, so the tests shipped with a given kernel version should always run well with that kernel. 

[Your editor thanks the Linux Foundation, LWN's travel sponsor, for supporting travel to this event.]
