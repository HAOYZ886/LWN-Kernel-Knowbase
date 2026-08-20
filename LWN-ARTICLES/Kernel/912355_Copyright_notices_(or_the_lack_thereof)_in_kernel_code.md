---
title: Copyright notices (or the lack thereof) in kernel code
url: https://lwn.net/Articles/912355/
date: "October 27, 2022"
category: Copyright issues
author: "By Jonathan Corbet October 27, 2022"
---

By **Jonathan Corbet**  
October 27, 2022

The practice of requiring copyright assignments for contributions to free-software projects has been in decline for years; the GNU Binutils project may be [the latest domino to fall](<https://sourceware.org/pipermail/binutils/2022-October/123680.html>) in that regard. The Linux kernel project, unlike some others, has always allowed contributors to retain their copyrights, resulting in a code base that has widely distributed ownership. In such a project, who owns the copyright to a given piece of code is not always obvious. Some developers (or their employers) are insistent about the placement of copyright notices in the code to document their ownership of parts of the kernel. A series of recent discussions within the Btrfs subsystem, though, has made it clear that there is no project-wide policy on when these notices are warranted — or even acceptable. 

In early September, a [patch series](<https://lwn.net/ml/linux-btrfs/cover.1662420176.git.sweettea-kernel@dorminy.me/>) implementing fscrypt integration for the Btrfs filesystem included [this patch](<https://lwn.net/ml/linux-btrfs/685c8abce7bdb110bc306752314b4fb0e7867290.1662420176.git.sweettea-kernel@dorminy.me/>) adding, among other things, a one-line Facebook copyright notice. Btrfs maintainer David Sterba [replied](<https://lwn.net/ml/linux-btrfs/20220909101521.GS32411@twin.jikos.cz/>) with a request to limit copyright information to SPDX tags; he cited [a page in the Btrfs wiki](<https://btrfs.wiki.kernel.org/index.php/Developer%27s_FAQ#Copyright_notices_in_files.2C_SPDX>), asserting that these tags are a complete replacement for copyright notices. Christoph Hellwig [disagreed](<https://lwn.net/ml/linux-btrfs/Yxs43SlMqqJ4Fa2h@infradead.org/>), pointing out that SPDX describes licensing but not ownership: 

> It is not a replacement for the copyright notice in any way, and having been involved with Copyright enforcement I can tell you that at least in some jurisdictions Copyright notices absolutely do matter. 

Hellwig, of course, was the initiator of a GPL-infringement lawsuit against VMware that was [dismissed](<https://lwn.net/Articles/696764/>) due to an inability to prove ownership of the code in question. It is thus unsurprising that he is sensitive to the placement of copyright notices in the code itself. When Hellwig submitted [a patch of his own](<https://lwn.net/ml/linux-btrfs/20220912141121.3744931-2-hch@lst.de/>), also in September, that added a copyright notice to a newly created file, Sterba [let it be known](<https://lwn.net/ml/linux-btrfs/20220921094014.GB32411@twin.jikos.cz/>) that he would refuse that change as well. Toward the end of October, in the discussion of [yet another patch set](<https://lwn.net/ml/linux-fsdevel/20220901074216.1849941-1-hch@lst.de/>), Hellwig eventually [withdrew the work](<https://lwn.net/ml/linux-fsdevel/20221024144411.GA25172@lst.de/>), saying: 

> FYI, I object to merging any of my code into btrfs without a proper copyright notice, and I also need to find some time to remove my previous significant changes given that the btrfs maintainer refuses to take the proper and legally required copyright notice. 

Given that the kernel code has no shortage of copyright notices (nearly 79,000 lines contain the word "copyright"), it is natural to wonder why this policy is being applied in the Btrfs subsystem. The Btrfs wiki page describes the reasoning: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

> The copyright notices are not required and are discouraged for reasons that are practical rather than legal. The files do not track all individual contributors nor companies (this can be found in git), so the inaccurate and incomplete information gives a very skewed if not completely wrong idea about the copyright holders of changes in a given file. The code is usually heavily changed over time in smaller portions, slowly morphing into something that does not resemble the original code anymore though it shares a lot of the core ideas and implemented logic. A copyright notice by a company that does not exist anymore from 10 years ago is a clear example of uselessness for the developers. 

The page also states that the Signed-off-by tags found in the kernel's Git history are sufficient to document the copyright status of the code. There are a few difficulties with this position, including the fact that those tags indicate that the submitter has the right to contribute the code to the kernel, but do not necessarily show who the copyright owner is. Another problem was [pointed out](<https://lwn.net/ml/linux-btrfs/YyfNMcUM+OHn5qi8@ebb.org/>) by Bradley Kuhn: if the Git history serves as the copyright notices for the code, then it will be necessary to ship the entire Git repository to be in compliance with the GPL's source-code requirements. That makes complaints about copyright notices in the code being unwieldy lose some of their weight. 

In the most recent discussion, Chris Mason [said](<https://lwn.net/ml/linux-fsdevel/773539e2-b5f1-8386-aa2a-96086f198bf8@meta.com/>) the ""Christoph's request is well within the norms for the kernel"". Sterba [replied](<https://lwn.net/ml/linux-fsdevel/20221024171042.GF5824@suse.cz/>) that he would consider changing the policy, but only as part of a wider policy decision by the kernel project: 

> I've asked for recommendations or best practice similar to the SPDX process. Something that TAB can acknowledge and that is perhaps also consulted with lawyers. And understood within the linux project, not just that some dudes have an argument because it's all clear as mud and people are used to do things differently. 

It's not clear who Sterba has asked for recommendations at this point. Chances are that he will find, over time, that the Btrfs subsystem's position on copyright notices is not widely held across the project as a whole. Steve Rostedt arguably [described the consensus view](<https://lwn.net/ml/linux-fsdevel/20221026074145.2be5ca09@gandalf.local.home/>): ""The policy is simple. If someone requires a copyright notice for their code, you simply add it, or do not take their code"". In the absence of a decree from Linus Torvalds, though, the issue of copyright notices may continue to be a source of disagreement. Claiming copyright on a portion of a shared body of code can always be a touchy matter, but it's one that developers can care a lot about.
