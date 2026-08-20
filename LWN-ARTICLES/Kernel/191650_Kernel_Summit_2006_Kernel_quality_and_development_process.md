---
title: "Kernel Summit 2006: Kernel quality and development process"
url: https://lwn.net/Articles/191650/
date: "July 18, 2006"
category: Development model
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
Andrew Morton ran a session to talk about kernel regressions and the development process in general. As has been documented elsewhere, Andrew feels that the quality of kernel releases is in decline and that more attention needs to be paid to tracking down and fixing bugs. He was not entirely successful in convincing the other developers of this, however. 

The talk began with the [survey of LWN readers](<http://lwn.net/Articles/190678/>) on kernel quality. Andrew called out a few details from the responses, including the fact that some 70% of respondents have encountered bugs in 2.6 kernels, but only 13% went on to say that the quality of the kernel is getting worse. Nearly half of the people who responded to the question feel that new features are emphasized too much over stability and bug fixing. Many respondents feel that problems reported to mailing lists do not get enough attention, and the majority of them feel that way about bugs reported via the kernel bugzilla. 

The first thing to do was to try to decide whether the results from the LWN survey actually meant anything. Surveys can be subject to no end of problems; is the LWN survey sane? After some discussion, the consensus was that perhaps the numbers make sense, but only after applying some filters. It was suspected that people who have had problems with 2.6 kernels were more likely to respond than those who are simply happy, so the overall level of problems reported is likely to be too high. In the experience of most kernel developers, many of the things reported as bugs are, in fact, simply unsupported hardware. Others are in fact hardware difficulties unique to a single system and unfixable in the kernel. 

Linus noted that, given the likely existence of selection bias, the results were relatively encouraging. Even people who have filled out a survey based on a bad experience tend, for the most part, to not feel too bad about the process as a whole. He was interested in the low vote of confidence in the kernel bugzilla, however. Ingo Molnar noted that fixing problems out of bugzilla is a low-visibility process - it feels like work, and is relatively unappealing. A lot of developers also just don't pay attention to it. One change which might come out of the discussion is automatically forwarding bugzilla reports (initial reports only) to the appropriate subsystem-specific mailing lists. 

James Bottomley, the SCSI maintainer, reported that the frequency of SCSI bug reports has remained about the same over the last few years. Greg Kroah-Hartman said that USB bugs were, if anything, dropping off a bit. Len Brown said that the number of ACPI bug reports remained about the same, even though the number of ACPI users is growing significantly; from that, he concludes that the code is getting better. 

The bottom line is that the kernel developers feel that there is no systemic quality problem in the development process, and that major changes are not needed. 

There are always improvements to be made, of course. Triaging and tracking bugs is a big job which takes too much of Andrew's time, and getting developers to fix bugs remains a problem. Much of the bug tracking work is an (approximately half time) job which could be shoved off onto somebody else, especially if said somebody else were to occupy the cubicle next to Andrew's. He is working on making that happen; it will be interesting to see how he brings this change about. There are a number of techniques which can be used to increase attention to bugs, such as public flaming, blocking the inclusion of other patches, etc. But Andrew described such measures as "childish," and not something which should be necessary in a community of professionals. He would, however, like to see fewer "it's not my problem" responses. Many bugs are hard to track down, and not all subsystems have active maintainers; developers should look beyond their own specific areas when helping to fix bugs. 

The -mm tree was discussed for a while. This tree is seen as perhaps breaking too often, a result of "dubious" patches being merged by Andrew and by subsystem maintainers. Andrew pledged to do better, and requested that the other developers do the same. There was discussion of creating a separate tree dedicated to patches intended for the next mainline merge cycle, but that idea was shot down. It was thought that such a tree would serve mainly to distract attention from both -mm and the mainline -rc releases. 

It was noted that developers are not spending enough time reviewing others' code. Beyond that, reviewers should take a more active role in helping developers to make their code ready for inclusion. The example raised here was the reiser4 filesystem, which has languished in -mm for two years. There's plenty of blame to hand out to the reiser4 developers, but they remain people who have put in a great deal of work trying to create interesting new functionality for the Linux kernel. We should be helping them to finish that job and get their code merged. 

Andrew asked: who is the i386 maintainer? According to the kernel maintainers file, it's Linus, but he's clearly not doing that at this point. Linus said that he sees i386 as being a legacy architecture in five years, and that ongoing maintenance could be a problem. His suggestion was to merge it into x86-64 as a sort of special case - making Andi Kleen its maintainer in the process. Andi showed a striking lack of enthusiasm for this idea. 

The problem of code entering the mainline without review by way of subsystem git trees was pondered for a bit. This practice was blamed for, among other things, the much-lamented merging of the wireless extensions netlink interface, which, at the summit, has received a fair amount of rather belated criticism. Linus suggested that each git tree maintainer should, as a stable kernel release approaches, post a textual description of the patches queued to be merged when the next kernel cycle starts. Others could then read those descriptions and have a good idea of what is coming; they could also choose to further investigate anything which looks questionable. It seems like a good idea which may not actually be implemented as often as one might like. 

A related problem is that of half-baked kernel patches being shoved into the mainline kernel because the only alternative is to miss the merge window and wait a few months. One possible outcome here might be a policy that all patches must be in the -mm tree before a kernel cycle begins to be considered for merging that time around. 

  * [Next: the `ioctl()` interface](<https://lwn.net/Articles/191653/>).
