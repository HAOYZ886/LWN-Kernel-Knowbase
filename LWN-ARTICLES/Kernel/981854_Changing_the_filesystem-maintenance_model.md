---
title: "Changing the filesystem-maintenance model"
url: https://lwn.net/Articles/981854/
date: "July 17, 2024"
category: "Development model-Maintainers"
author: "By Jake Edge July 17, 2024 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
July 17, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Maintenance of the kernel is a difficult, often thankless, task; how it is being handled, the role of maintainers, burnout, and so on are recurring topics at kernel-related conferences. At the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Josef Bacik and Christian Brauner led a session to discuss possible changes to the way filesystems are maintained, though Bacik took the lead role (and the podium). There are a number of interrelated topics, including merging new filesystems, removing old ones, making and testing changes throughout the filesystem tree, and more. 

#### New model?

The current model for filesystem maintenance—"that we all know; we love it"—is that each filesystem has its own tree and pull requests are sent to Linus Torvalds directly, Bacik began. If changes to the virtual filesystem (VFS) layer are needed, those are sent to Brauner or Al Viro depending on what part of the VFS they touch. This works well for established filesystems, but it can run into problems when there are changes that cross filesystem boundaries, such as folios, the new mount API, namespace things, etc. That kind of work is happening more frequently, and gets discussed at LSFMM+BPF, but many of the plans made do not come to fruition, perhaps in part because "nobody feels empowered to make those decisions and push that work through". 

[ ![\[Josef Bacik\]](https://static.lwn.net/images/2024/lsfmb-bacik-sm.png) ](<https://lwn.net/Articles/982095/>)

There is also "a little bit of vagueness about who owns what"; in particular, merging new filesystems can go smoothly or can be a "painful contentious thing" where the decision is left to Torvalds. That is not necessarily a bad thing, but it does create uncertainty for submitters of new filesystems; it can also lead to the concerns of experienced developers being waved away when Torvalds tires of the arguments and simply merges the filesystem. It is a strength of the Linux community that Torvalds can step in and just merge things, but it means that new filesystem developers need to be persistent and to try to sway Torvalds to ignore the other voices in order to get their code merged, Bacik said. 

For most of a year now, he has been proposing that the filesystem community in Linux move toward a model like those of some other kernel subsystems, notably networking and graphics. In both cases, they have multiple, disparate pieces that all funnel through the same maintainer and tree. That can clearly be seen in the [eye chart](<https://lwn.net/Articles/981742/>) that accompanies our [6.10 development statistics article](<https://lwn.net/Articles/981559/>). Bacik thinks that changing in that way would help remove some of the pain points when making changes across the filesystems. 

When tree-wide changes are made, currently the developer needs to go figure out where the right trees are for each filesystem, which can be painful. In fact, he said, Btrfs is probably one of the worst offenders in that regard, because its tree is on GitHub; its policies and procedures are different from the others as well. Recently, Christoph Hellwig has been doing some work in Btrfs and told Bacik that those differences are irritating enough that it makes him want to avoid working on Btrfs. 

Overall, Bacik feels that, because of those differences, the filesystem community is more difficult to deal with than it should be, especially given how much code is shared between the filesystems. So he was proposing the creation of a single tree that all of the filesystems push into; pull requests would no longer be sent to Linus by the individual filesystem maintainers, but would come from the maintainer of that tree. That is mostly a mechanical change, but he would like to see some cultural changes as well. 

Viro noted that there are lots of filesystems in the kernel tree that do not use VFS; they are specialized filesystems that live in the `security` and `arch` trees, among other places. He wondered if Bacik really wanted to have filesystems for s390 and ppc (PowerPC), for example, live in this new tree. Bacik said that those were not his focus; he is after the filesystems that need changes like folio support, converting to the new mount API, switching to iomap, and so on. 

But Viro pointed out that the new mount API affects all filesystems, including, say, the USB gadget filesystems. Bacik said that was a good point, which invalidated some of what he was suggesting. But his goal is to make it easier for people who are doing cross-filesystem work to "get their patches merged without having to understand how each individual filesystem community works". 

Viro suggested having a single topic branch that was shared for, say, folio work. That works for VFS topic branches, Bacik said, because all of the filesystems are going to pay attention to those. Every time there needs to be a branch for non-VFS changes, though, it requires a negotiation of whose tree that will all flow through, he said, which he would like to try to avoid. 

#### An fs-next?

Dave Chinner gave a few-day-old example of this kind of problem. Some changes to iomap that caused an XFS regression had gone into Brauner's tree. It was only discovered because of testing that was being done on linux-next, which is where XFS and the iomap changes met. Earlier, iomap changes were flowing through the XFS tree, so this problem would have been caught sooner. Since the merge window was happening while the conference was going on, that bug had probably already been merged into the mainline, Chinner said. 

So the changes that were recently made to try to "centralize some of the infrastructure up into VFS trees is actually backfiring on us", Chinner said. That is because the next step of being able to share those changes down to the individual filesystems for testing is not being done. What he would like to see is an fs-next tree that is a combination of all of the filesystem trees that are planned for the next release; it would become the common test base that everyone uses for regression testing, but would not be pushed as a whole into the mainline. 

Someone in the audience asked why linux-next could not be used. Chinner said that it is not reliable enough for use as a test base. Fs-next would have better reliability, Chinner said in answer to the obvious followup, because it would not have "device failures and all sorts of driver problems". Historically, linux-next has not been reliable enough to use for regression testing or day-to-day development, he said. 

Shifting gears a little, Brauner said that he does not think that all of the filesystems need to flow through a single tree that gets passed to Torvalds for merging. That is infeasible, in his mind, both technically and socially. A lot of the people in the room probably enjoy the fact that they get to send pull requests to Torvalds, he said. A better solution would be to have an fs-next tree. 

Ted Ts'o said "linux-next is not terrible", but that, in practice, he waits until -rc6 or -rc7 before he does a test merge of the ext4-development branch into the mainline -rc tree for integration testing. The major problems that occur have generally been worked out by -rc6 or -rc7; "somebody else has taken the arrows first". Sometimes that falls by the wayside, though, for example when it those -rc releases happen just before a conference like LSFMM+BPF, which is a downside to waiting that long for integration testing, he said. 

No matter whether it is done using linux-next or fs-next, Ts'o said, the important thing is for filesystem communities to be testing more than just their own branch. It is a big change, though, that needs to be centralized in order to work, Brauner said. He has his set of tests that he runs on changes to the VFS infrastructure that he picks up, but "I have zero idea what the XFS test matrix is and it is probably a _bit_ larger". There is no centralized testing, so "we don't actually know if what we are doing is not breaking anyone". 

Chinner said that each filesystem is in its own silo, but that there is a need to test the common code. Neal Gompa said that each filesystem had its own mechanisms for testing, reporting, coverage calculation, etc., but there is no reason those all could not run on a system where people could push code to give them a reasonable level of confidence that they did not break things. 

#### Common testing infrastructure

Matthew Wilcox thought that much of this problem could be solved by talking to linux-next maintainer Stephen Rothwell. It would probably not be difficult for him to merge all of the filesystem trees into a separate branch. But even before a common branch gets created, there needs to be common testing infrastructure, Kent Overstreet said. It would only require a couple of racks of hardware to set up; he has been hoping not to have to do that himself, but may have to. (As of June 14, he had [started work](<https://lwn.net/ml/all/o55vku65ruvtvst7ch4jalpiq4f5lbn3glmrlh7bwif6xh6hla@eajwg43srxoj/>) on a shared filesystem test cluster.) When branches were pushed to it, all of the tests for all of the different filesystems would run and the system would provide a "simple dashboard" to report results. The current email-based system does not work, he said, since he cannot determine which tests are being run. 

Bacik said that he has an automated system for testing Btrfs; he could add, say, tests for XFS, bcachefs, and others to that. But there is a lot of work that the Btrfs developers do to maintain the set of tests that should be excluded (and the reasons why) from a run of [fstests](<https://lwn.net/Articles/897061/>). That is done to ensure that the test results are usable. Someone would need to do that work for the other filesystems. 

Chinner said that there is no need for Btrfs developers to test XFS or other filesystems. The important thing is for filesystem developers to test their filesystem "with the same tree" to shake out integration problems with the cross-filesystem work that is happening. So, all of the latest code for each filesystem would be in the same tree for testing by the various communities using their own tests and infrastructure. That means that the tree would be integrated, rather than trying to integrate all of the different testing infrastructure that each filesystem has built up. 

There is a need to be able to "go somewhere and click on something" to see that a change "worked for everything that is relevant", Brauner said. Gompa agreed with Chinner that the existing infrastructure could be used, but the testing could be done against a single tree, such as fs-next, that filesystem developers use as the basis for (and eventual destination of) their work. The automated-testing systems that each filesystem uses would also be based around the integration tree; perhaps a dashboard that collects the results from each individual filesystem's infrastructure could be developed. Eventually, moving everyone to the same testing infrastructure, with a single integrated dashboard and reporting mechanism, could be worked on. The idea would be to take "baby steps" toward a world where doing cross-filesystem development (or even just following it, as he does) is "a hell of a lot easier". 

Instead of testing a for-next branch in, say, the XFS tree, the test target should be the fs-next tree, Chinner said; nothing else changes in the development workflow or in the actual tests that are run. Ts'o said that there may still be tests of various other branches as part of the filesystem development, but that the fs-next tree would be tested daily, say, by each of the filesystems. 

David Howells asked if there could be a common base with the iomap changes, folio-support changes, and the like for use by the filesystems. Viro said that could be created, but it would require agreement on exactly which trees are included. Howells wondered about how pull requests would be done, since they cannot be based on fs-next. There would still need to be development trees that are maintained so that pull requests can be made to Torvalds, Chinner said, but that code will have been tested with the integration tree to head off problems before they reach Torvalds. 

There was some scattered conversation, largely not using the microphones, about exactly which patches should go where, and when, to try to avoid problems with bugs reaching the mainline that should have been caught earlier. Brauner said that io_uring is often a place where changes that he makes affect a completely different tree; he just creates a stable branch with those changes that Jens Axboe pulls into his tree for testing. Some of that will still need to happen, but the problem with the iomap changes that was mentioned by Chinner was an example of where fs-next would have helped. 

Bacik closed the session by noting that he would still like to figure out how to make the lives of cross-filesystem developers easier so that they did not have to try to figure out how to integrate with a bunch of different filesystem-development communities. But that would have to wait since the session was out of time. 

A [YouTube video](<https://www.youtube.com/watch?v=T0secw6AkzQ>) of the session is available.
