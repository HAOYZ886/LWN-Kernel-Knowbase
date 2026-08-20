---
title: Policies for merging new filesystems
url: https://lwn.net/Articles/1074557/
date: "May 28, 2026"
category: Filesystems
author: "By Jake Edge May 28, 2026 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 28, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

In a filesystem-track session at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Amir Goldstein wanted to discuss his [proposed documentation on adding new filesystems](<https://lwn.net/ml/all/20260417142503.1436446-1-amir73il@gmail.com/>) to the kernel. There are a number of unmaintained and untestable filesystems already in the kernel, which are a burden to VFS-layer developers who are trying to make sweeping changes, such as switching to folios and the "new" mount API. Goldstein's document is an attempt to head off the addition of filesystems that may increase that burden down the road. 

The idea behind the document is to provide prospective filesystem projects a bit of a checklist for what is needed in order to be considered for inclusion. Two new filesystems, [VMUFAT](<https://lwn.net/ml/all/20260410141723.307925-2-adrianmcmenamin%40gmail.com/>) and [FTRFS](<https://lwn.net/ml/all/20260413142357.515792-1-aurelien@hackers.camp/>), were posted in the same week that he had proposed the document. Both of those projects needed to be asked the usual questions (such as "why not FUSE?"), pointed toward the need for adding fstests, and so on. Getting all of that down in one place that developers of new filesystems can be referred to will help. 

#### Criteria

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2026/lsfmb-goldstein-sm.png) ](<https://lwn.net/Articles/1074903/>)

The filesystem developers need to decide what message they want to send to prospective new filesystems; he noted that the rejection of [ZUFS](<https://lwn.net/Articles/756625/>) sent a message, but he wondered if the community's message had changed since then. Christian Brauner said that for a few years there were multiple different container filesystems that were proposed, but that he never believed a single filesystem would work for all of the different kinds of containers the kernel supports. Pushing back under those circumstances makes sense, but there may come a time when there is a real need to add another filesystem to the kernel. ""I just think the bar should be quite high to do that."" 

The community has seen the pain of bringing a filesystem in and needing to reverse course, Brauner said. There is a balance that needs to be found and criteria for new filesystems can help with that. Part of the problem, though, is that new filesystems are generally sent to Linus Torvalds for a decision; the filesystem community does not necessarily get a veto. 

Ted Ts'o pointed out that the interest in using FUSE for filesystems has increased over the last 12-18 months. Much of that was driven by the security concerns with in-kernel filesystems. He pointed to the work that Darrick Wong has been doing on [implementing ext4 as a FUSE filesystem](<https://lwn.net/Articles/1044432/>) that can still perform at close to native speed; that was aimed at providing better security for container workloads. Ts'o does not consider containers to provide a security boundary, but some who do are interested in accessing ext4 filesystems that way. 

He does not think that the filesystem community should be pushing new filesystems to use FUSE, however. The bar should be set high enough for a new filesystem ""so that it is long-term maintainable by the community"". The new filesystem developers can then decide what approach they want to take. 

An attendee suggested having a list that described the reasons why a filesystem should be in the kernel, as well as why it should not be. There are some existing in-kernel filesystems that do not need to be there, he said. The problem with that is that ""someone has to write it"", Jeff Layton said to laughter. 

Damien Le Moal agreed with all of the criteria, such as needing tests and maintainers, calling it ""a no brainer"" for any proposed feature. In the special case of a filesystem, though, he said that the first question that the project needs to answer is "why?": ""what is the exact thing you cannot do"" with the existing choices? ""What problems is it solving?"" Without answers to those, there is no point in accepting a filesystem, no matter how good the code is or how committed the maintainers are. 

#### Removal

It is extremely difficult to remove a filesystem once it has been merged, Josef Bacik said; addressing that is where the group should spend its time. The criteria for inclusion is reasonably well-known, but the process to deprecate and remove one is not. So it is not just criteria for merging that needs to be documented, but filesystem projects also need to continue with a certain maintenance level and keep up with the rest of the community, which should be described. If a filesystem falls behind, ""this is how we are going to get rid of your filesystem"". 

Brauner said that there had been [pushback](<https://lwn.net/Articles/951846/>) from Greg Kroah-Hartman and others at a Maintainers Summit about leaving obsolete filesystems in the tree, as is done with drivers. There are drivers that ""just sit and rot"", perhaps filesystems should be treated the same way, they suggested. Jan Kara wondered if it made sense to simply implement a FUSE version of some of these unwanted filesystems as a way to be rid of them, though regressions might get in the way of the final step. 

Brauner noted that he had put out a policy statement that bugs from corrupted filesystem images were not considered security problems. If that were not the case, there would be an unending stream of CVEs that would need to be addressed, particularly for older filesystems. He said that the [flood of bug reports coming from LLMs](<https://lwn.net/Articles/1066581/>) has led to [pulling out some kernel code](<https://lwn.net/Articles/1068928/>) from the networking subsystem, so ""removal, suddenly, is something that _can_ be done, apparently, without much objection"". 

Ts'o said that historically, ""really simple"" filesystems that do not use ""any of the advanced interfaces"" can remain in the tree and "rot" without causing much in the way of problems. There are certainly bugs lurking in some of those, which could be found by fuzzing or LLMs, ""but nobody cares"". The community could be explicit about that and simply list legacy filesystems with disclaimers like ""no one cares about them from a security perspective, if you're a distro, we strongly recommend you don't build the suckers"". 

It makes sense to send a strong signal that some are second-class filesystems, he said. ""The only reason why we haven't just simply created a FUSE driver and then ejected the in-kernel version is that it's not even worth the effort to do that."" The list may be the lowest energy fix to the problem while allowing the few users of those filesystems to keep doing so. It is similar to support for the PA-RISC architecture in the kernel, which almost certainly has security problems of various sorts, but the kernel community seems unconcerned. 

There was some fast-moving discussion around marking filesystems as deprecated or listing them as Ts'o suggested, though no real conclusion was reached. Meanwhile, Goldstein wondered if new filesystems should be required to be maintained out of the tree for a while as a proving ground for the code as well to show that there are interested users. That has been a successful path for some filesystems in the past. 

#### Famfs

Goldstein raised the [discussion about merging famfs](<https://lwn.net/Articles/1068686/>). He said that there was never any real pushback against merging famfs in some form, it largely came down to whether it should use FUSE or not. Those questions allowed its developer, John Groves, to try both to see which made more sense; it was a kind of extended ""design review"", Goldstein thought, which was valuable as part of the process. 

Groves did not disagree, though he hoped it would not be another year before famfs was merged in some form. There is a bit of an impedance mismatch since famfs uses memory, not storage, so data is not really persistent. But famfs is also low risk as a new kernel filesystem, since it cannot be used as a general-purpose filesystem. It is aimed as a solution to a new problem that comes about with a, say, multi-TB memory appliance. 

Goldstein agreed, but noted that the process was meant to see if something more generalized could come out of it, which would make it a better fit for the upstream kernel. Groves said that two years ago he had [presented](<https://lwn.net/Articles/983105/>) the filesystem at LSFMM+BPF and switching to FUSE was suggested; he was resistant at first, but returned to the summit in 2025 [with a FUSE-based implementation](<https://lwn.net/Articles/1020170/>). He maintains both versions and is leaning toward suggesting a return to the standalone filesystem for merging. 

Many of the requirements that are being suggested for new filesystems, such as working tests in fstests, only really apply to filesystems that have a persistent on-disk format, Ts'o said. That's important because there need to be different standards for, say, a new read-only or network filesystem. For example, EROFS is being used in ways that are ""beyond its original use case, which is a sign that the people who did it got it right"". It is unlikely that anyone will take on the pain of creating a new read-only or network filesystem unless there is truly a need; ""it's just not worth the effort"". The criteria for those, if they were to appear, are sure to be different than those in the document, which are geared toward regular, persistent on-disk filesystems. 

[I would like to apologize for any errors here. The acoustics in the room were problematic for both hearing and recording. Misunderstanding and misidentification may have resulted.]
