---
title: Analyzing kernel email
url: https://lwn.net/Articles/804511/
date: "November 13, 2019"
category: "Development model-Email analysis"
author: "By Jake Edge November 13, 2019 ELCE"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
November 13, 2019

* * *

[ELCE](<https://lwn.net/Archives/ConferenceByYear/#2019-Embedded_Linux_Conference_Europe>)

Digging into the email that provides the cornerstone of Linux kernel development is an endeavor that has become more popular over the last few years. There are some practical reasons for analyzing the kernel mailing lists and for correlating that information with the patches that actually reach the mainline, including tracking the path that patches take—or don't take. Three researchers reported on some efforts they have made on kernel email analysis at the [2019 Embedded Linux Conference Europe](<https://events19.linuxfoundation.org/events/embedded-linux-conference-europe-2019/>) (ELCE), held in late October in Lyon, France. 

The presentation ([slides [PDF]](<https://static.sched.com/hosted_files/osseu19/5c/pasta-elce19.pdf>)) actually listed four speakers, though one could not make it to ELCE. The three present were Ralf Ramsauer, from the Technical University of Applied Sciences Regensburg, Sebastian Duda, of Friedrich–Alexander University Erlangen–Nürnberg, and Wolfgang Mauerer, of Siemens AG in Munich. Lukas Bulwahn, who is a hobbyist active in the Linux Foundation [ELISA Project](<https://elisa.tech/>) and employed at BMW AG, was unable to attend. In the introduction, Mauerer jokingly suggested that the goal of the research was to understand more "than the NSA already knows" about the behavior of kernel developers. Really, though, the presentation was meant partly as a request for comments; the researchers have been observing the kernel community for some time and have been pulling out pieces they find interesting, but they would be happy to hear other ideas on the kinds of analysis that would be useful to the community. 

#### Development process

[ ![\[Wolfgang Mauerer\]](https://static.lwn.net/images/2019/elce-mauerer-sm.jpg) ](<https://lwn.net/Articles/804678/>)

Ramsauer said that the goal of the research is "formalizing and assessing the Linux kernel development process". There are a number of motivations for doing so, both from inside and outside the community; his personal motivation is to write more papers to finish his PhD work, he said with a grin. There is interest from the safety-critical development world, automotive and industrial equipment makers, for example. The safety-certification bodies require documented development practices; since the development process of Linux is not something under the control of the equipment makers, the best they can do is to document the existing practice to those bodies. Beyond that, the techniques they are working on can be used to monitor the health of Linux and other projects, as is done by the [CHAOSS project](<https://chaoss.community/>). 

Tracing the development process has come up recently in the kernel community as well. The interest in tracking patches through various stages in their development was the underlying motivation for the proposal to [add change IDs to patches](<https://lwn.net/Articles/797613/>). There were alternatives proposed, such as adding the Message-ID of the previous patch posting to an update, but that is not consistently done throughout the kernel. The community would like to find a solution to this problem that avoids the, perhaps only minor, inconvenience of maintaining those IDs. 

The problem stems from the disconnect between activity on the mailing list and commits in the Git repositories. He went through the typical lifecycle of a patch set, starting with its development, which is done in private typically on some Git branch. Eventually the patch set surfaces on one or more kernel mailing lists, where it is reviewed. The review comments are reflected in the patches and they are posted again. That process is followed, iteratively, until the patches are acceptable and merged (or not). 

[ ![\[Ralf Ramsauer\]](https://static.lwn.net/images/2019/elce-ramsauer-sm.jpg) ](<https://lwn.net/Articles/804680/>)

Each posting of the patch set, along with all of the comments and discussion associated with it, can be identified by the Message-ID in the posting, which is new for each iteration. But the commit (or commits) that stem from the patch set are identified with a Git commit hash. There is potentially a many-to-many relationship between these message IDs and commit hashes, which leads to a need for a tool to extract that information from the mailing lists and Git repositories. 

They have a tool, [Patch Stack Analysis](<https://github.com/lfd/PaStA>) (PaStA), that can do that work. It originally was written to detect _similar_ patches between different branches in order to quantify the upstreaming efforts of various out-of-tree projects (e.g. vendor kernels, realtime patches). It has now been extended to work with mailing lists. 

He gave an example of two patches to show how they are processed to determine how similar they are. The patches are tokenized and then string distance measurements are used to generate a similarity score. If that score exceeds a threshold, the patches are considered likely to be related and a similarity graph is created, which gets matched up with patches in the repository. More information about the techniques used can be found in two papers ([PDF 1](<https://arxiv.org/pdf/1902.03147.pdf>), [PDF 2](<https://arxiv.org/pdf/1607.00905.pdf>)) they have written, Ramsauer said. 

The Git data to be used is easy to come by simply by cloning the repository of interest, but the mailing list data is not as easy. Up until a few years ago, they used the data from Gmane, but that has become unreliable after the [demise of Gmane](<https://lwn.net/Articles/695695/>) (mostly) in 2016. [Lore.kernel.org](<https://lore.kernel.org/lists.html>) is a reasonable substitute, with some data going back to 1996 and beyond, but it has a limited subset of mailing lists. It also has imported some of its data from Gmane, which improperly handled some headers, so it can be difficult to work with. 

Due to that, they started collecting their own [archive of around 200 kernel mailing lists](<https://github.com/linux-mailinglist-archives>) in May. In doing so, they ran into the same problems that anyone dealing with a large pile of email encounters: email is full of all sorts of weirdness. There are broken encodings, bad dates, badly formatted Message-ID headers, and more. Once that all gets cleaned up, that data will be ready to use, he said. 

#### Some results

Duda then stepped up to the podium to report on what had been found in their analysis. They looked at the roughly 610,000 commits to the mainline starting with the v2.6.39 tag along with around three-million emails from the public lists on lore.kernel.org; 34 mailing lists were used, starting in May 2011 and running until the end of 2018\. Of the three-million emails, though, not all are patches; eliminating the non-patches left 1.15m patches. But not all patches posted to the lists are actually patches to the kernel; beyond that, there is a fair amount of traffic that comes from bots, pull requests, stable reviews, and the like. In the end, they winnowed it all down to around 800,000 emails with patches from humans that were meant to be applied to the current kernel at the time they were posted. That is what they ran through the analysis program. 

[ ![\[Sebastian Duda\]](https://static.lwn.net/images/2019/elce-duda-sm.jpg) ](<https://lwn.net/Articles/804679/>)

They set out to see if they could identify unmaintained subsystems based on patches that get ignored on the mailing lists. As it turns out, they were not able to do so because the two do not correlate. In the process, though, they found other interesting things. First they defined an ignored patch as one where the posting thread has no responses other than by the author, where the patch itself does not get accepted upstream, and that any related (similar) patches for, say, other kernel versions were also ignored. 

Based on that criteria, 2.5% of patches were ignored over the time period. Interestingly, the percentages seemed to drop over time: 2011 had 3.9%, 2015 had 2.1%, and 2018 had 1.6%. When graphing that, which can be seen in slide 18 or around 17:50 in the [YouTube video](<https://www.youtube.com/watch?v=YCfU-2dXDq0>) of the talk, a large spike appears in the ignored patch line midway through 2016. That comes from a single 1300-patch posting that, perhaps unsurprisingly, got ignored. Since it throws off the rest of the graphs, they simply eliminated that outlier in subsequent graphs. 

What the corrected graph shows is that there are roughly 30-50 patches that are ignored each week throughout the time frame. But the number of patches submitted each week rose over that same time frame, which leads to the percentage decline. At a talk given at the [Linux Plumbers Conference](<https://www.linuxplumbersconf.org/>) (LPC), some maintainers asked for a way to find out how many patches they were ignoring. To try to help answer that question, Duda said that they have also run the tool on individual mailing lists. 

He presented four graphs of the linux-arm-kernel, linux-mips, linux-wireless, and netdev mailing lists. One thing that stands out right away is that all show a steady number of ignored patches. That number is quite low in all four, ten or less per week, and it remains flat even when the overall number of patches posted to the mailing list grows. 

It is interesting that over the time period, the number of patches to linux-kernel-arm went from around 150 per week to as many as 700 per week, but that the ignored patches remained below ten per week, Ramsauer said. That trend persists with most mailing lists, though there a few that look different, he said. They chose the linux-kernel-arm and linux-kernel-mips lists because both are architectures, but the Arm list has grown substantially over the years, while MIPS has stayed roughly the same. Likewise for linux-wireless and netdev, though linux-wireless has actually declined slightly over the years. 

Another analysis that they did, Ramsauer said, is to try to see if the likelihood of being ignored is correlated to when in development cycle it is sent. As it turns out, being ignored is largely independent of when in the cycle a patch is sent. There is, however, a slightly higher chance of a patch being ignored during the merge window. 

#### Off-list patches

During this work, they also discovered some "off-list" patches, which have been included into Linus Torvalds's Git tree, but were not posted to a public mailing list prior to that. They analyzed the stabilization phase of the 5.1 kernel, patches from v5.1-rc1 to v5.1, which was around 1800 commits. They mapped those commits back to the mailing list and found 60 that did not have a mailing-list thread identified. Some of those were errors in their tool, but a manual review showed that 24 patches were off-list patches. 

Some of those were reverts, where doing the revert _was_ discussed on the list, but the actual patch doing the revert was not posted. Some were also simply fixups of various sorts by the maintainers that were never discussed. There are some subsystems where the maintainers often have off-list patches, he said. In addition, of course, some of those patches will be security fixes that were discussed, but not publicly. For example, he pointed to a [commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v5.4-rc7&id=c7084edc3f6d67750f50d4183134c4fb5712a5c8>) from Greg Kroah-Hartman that never appeared on a public list; when asked, Kroah-Hartman said that it had been discussed on the closed kernel security mailing list. Thus this kind of analysis might provide a way to find security fixes before they become publicly known, Ramsauer said. 

Several audience members were interested to know which subsystems tended to have off-list patches, but the researchers did not want to point fingers. It is entirely possible for others to do the same analysis, however, as an attendee pointed out, which could lead to a lucrative, if possibly illegal, "business" of disclosing them in various ways. The clear takeaway is that off-list patches are likely to become more visible, rather than quietly lurking in the sea of other patches, before too long. 

As Mauerer noted at the end of the talk, this kind of research is not well funded, either by industry or by normal academic research channels. Presenting results at a conference like ELCE is not deemed important in academic circles, while presenting it in a workshop to half-a-dozen people who have never submitted a single kernel patch is seen as valuable in that realm. He encouraged companies and other organizations to consider funding this kind of research in the future. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to attend Embedded Linux Conference Europe in Lyon, France.]
