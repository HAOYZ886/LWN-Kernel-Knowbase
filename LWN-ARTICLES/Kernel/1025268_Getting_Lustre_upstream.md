---
title: Getting Lustre upstream
url: https://lwn.net/Articles/1025268/
date: "June 18, 2025"
category: "Filesystems-Lustre"
author: "By Jake Edge June 18, 2025 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
June 18, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The [Lustre filesystem](<https://www.lustre.org/>) has a long history, some of which intersects with Linux. It was added to the staging tree in 2013, but was [bounced out of staging](<https://lwn.net/Articles/756565/>) in 2018, due to a lack of progress and a development model that was incompatible with the kernel's. Lustre may be working its way back into the kernel, though. In a filesystem-track session at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF), Timothy Day and James Simmons led a discussion on how to get Lustre into the mainline. 

Day began with an overview of Lustre, which is a ""high-performance parallel filesystem"". It is typically used by systems with lots of GPUs that need to be constantly fed with data (e.g. AI workloads) and for checkpointing high-performance-computing (HPC) workloads. A file is split up into multiple chunks that are stored on different servers. Both the client and server implementations run in the kernel, similar to NFS. For the past ten or more years, the wire and disk formats have been ""pretty stable"" with ""very little change""; Lustre has good interoperability between different versions, unlike in the distant past where both server and client needed to be on the same version. 

[ ![\[Timothy Day\]](https://static.lwn.net/images/2025/lsfmb-day-sm.png) ](<https://lwn.net/Articles/1025635/>)

The upstreaming project has been going on for a long time at this point, he said. A fork of the client was added to the staging tree and resided there for around five years before ""it got ejected, essentially due to insufficient progress"". It was a ""bad fit"" for the kernel, since most developers worked on the out-of-tree version, rather than what was in staging. 

But ""the dream of actually getting upstream still continued"". There have been more than 1000 patches aimed at getting the code ready for the kernel since it got ejected; around 600 of those were from Neil Brown and 200 came from Simmons. Roughly 1/3 of the patches that have gone into the out-of-tree repository since the staging removal have been related to the upstream goal, Simmons said. 

Day said that the biggest question is how the project can move from its out-of-tree development model to one that is based around the upstream kernel repository. The current state is ""a giant filesystem, #ifdef-ed to hell and back to get it working with a bunch of kernel versions"". The next stage, which is currently being worked on and is slated to complete in the next year or so, is to split the compatibility code out of the core filesystem code; the goal is to eventually have two separate trees for those pieces. The core filesystem tree would go into the kernel tree, while the compatibility code, which is meant to support customers on older kernels, would continue to live in a Lustre repository. 

Another area that needs attention is changes to the development process to better mesh with kernel development. The Lustre project does not use mailing lists; it uses a [Gerrit](<https://www.gerritcodereview.com/>) instance instead. ""We have got to figure out how to adapt."" Simmons said that there are some developers who are totally Gerrit-oriented and some who could live with a mailing list; ""we have to figure out how to please both audiences"". 

Amir Goldstein said that the only real requirement is that the project post the patches once to the mailing list before merging; there is no obligation to do patch review on the list. Simmons said that he and Brown have maintained a [Git tree](<https://github.com/jasimmons1973/lustre>) since Lustre was removed from staging; it is kept in sync and updated to newer kernels. All of the patches are posted to the [lustre-devel mailing list](<http://lists.lustre.org/listinfo.cgi/lustre-devel-lustre.org>) and to a [Patchwork](<https://patchwork.readthedocs.io/en/latest/>) instance, so all of the history is open and available for comments or criticism, he said. 

[ ![\[James Simmons\]](https://static.lwn.net/images/2025/lsfmb-simmons-sm.png) ](<https://lwn.net/Articles/1025637/>)

Josef Bacik asked about what went wrong when Lustre was in the staging tree. From his perspective, Lustre has been around for a long time and there are no indications that it might be abandoned, so why did it not make the jump into the mainline `fs/` directory? Simmons said that the project normally has a few different features being worked on at any given time, but Greg Kroah-Hartman, who runs the staging tree, did not want any patches that were not cleanups. So the staging version fell further and further behind the out-of-tree code. Bacik said that made sense to him; ""that's like setting you up to fail"". 

Christian Brauner said that he would like to come up with a ""more streamlined model"" for merging new filesystems, where the filesystem community makes a collective decision on whether the merge should happen. The community has recently been ""badly burned by 'anybody can just send a filesystem for inclusion' and then it's upstream and then we have to deal with all of the fallout"". As a VFS maintainer, he does not want to be the one making the decision, but merging a new filesystem ""puts a burden on the whole community"" so it should be a joint decision. 

Bacik reiterated that no one was concerned that Lustre developers were going to disappear, but that there are other concerns. It is important that Lustre is using folios everywhere, for example, and is using ""all of the modern things""; that sounds a little silly coming from him, since Btrfs is still only halfway there, he said. Simmons said that the Lustre developers completely agree; there is someone working on the folio conversion currently. At the summit, he and Day have been talking with David Howells about using his [netfs library](<https://www.kernel.org/doc/html/latest/filesystems/netfs_library.html>). 

Jeff Layton asked if the plan was to merge both the client and the server. Simmons said that most people are just asking for the client and that is a slower-moving code base. The client is ""a couple-hundred patches per month, the server is three to four times the volume of patches"", which makes it harder to keep up with the kernel. Layton said: ""baby steps are good"", though Day noted that it is harder to test Lustre without having server code in the kernel. 

The reason he has been pushing for just merging the client, Ted Ts'o said, is because the server is ""pretty incestuous with ext4"". The server requires a bunch of ext4 symbols and there is ""a need to figure out how to deal with that"". That impacts ext4 development, but other changes, such as the plan to rewrite the [jbd2 journaling layer](<https://www.kernel.org/doc/html/latest/filesystems/ext4/journal.html>) to not require [buffer heads](<https://docs.kernel.org/filesystems/buffer.html>), may also be complicated by the inclusion of the Lustre server, he said. 

Simmons asked about posting patches to the linux-fsdevel mailing list before Lustre is upstream so that the kernel developers can start to get familiar with the code. Bacik said that made sense, but that he would not really dig into the guts of Lustre; he is more interested in the interfaces being used and whether there will be maintenance problems for the kernel filesystem community in the Lustre code. Goldstein suggested setting up a Lustre-specific mailing list, but Simmons noted that lustre-devel already exists and is being archived; Brauner suggested getting it added to [lore.kernel.org](<https://lore.kernel.org/>)

The intent is that when Linus Torvalds receives a pull request for a new filesystem that he can see that the code has been publicly posted prior to that, Goldstein said. It will also help if the Git development tree has a mirror on [git.kernel.org](<https://git.kernel.org/>), Ts'o said. Bacik said that he thought it was a probably a lost cause to try to preserve all of the existing Git history as part of the merge, though it is up to Torvalds; instead, he suggested creating a git.kernel.org archive tree that people can consult for the history prior to the version that gets merged. 

Given that Lustre targets high performance, Ts'o said, it will be important to support large folios. Simmons said that someone was working on that, and that it is important to the project; the plan is to get folio support, then to add large folios. Matthew Wilcox said that was fine, as long as the page-oriented APIs were getting converted. Many of those APIs are slowly going away, so the Lustre developers will want to ensure the filesystem is converted ahead of those removals.
