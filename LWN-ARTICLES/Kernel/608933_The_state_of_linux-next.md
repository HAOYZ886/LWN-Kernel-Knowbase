---
title: "The state of linux-next"
url: https://lwn.net/Articles/608933/
date: "August 20, 2014"
category: "Development model-linux-next; linux-next"
author: "By Jonathan Corbet August 20, 2014 Kernel Summit 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 20, 2014

* * *

[Kernel Summit 2014](<https://lwn.net/Articles/KernelSummit2014/>)

Once the discussion of the stable tree had completed, the group at the 2014 Kernel Summit turned its attention to the most unstable tree of all: linux-next. This tree holds code that, for the most part, is destined to go into the mainline during the next merge window; its contents can change wildly over the course of a development cycle as patches are added and removed. Linux-next maintainer Stephen Rothwell provided an update on how this tree is doing. 

In general, Stephen said, about 90% of the code that shows up in the mainline during the merge window was in linux-next prior to the opening of the window. Is that good enough? He left that for the group to decide. He also noted that there are currently more than 220 trees feeding into linux-next, and wondered if that might be too many. About 25 of those, he said, are dedicated to urgent fixes and are probably OK. He is not so sure about the "preparatory trees" that feed patches into another tree that is also pulled into linux-next. Many of those trees are not as stable as he would like; the code found there may not really be ready for mainline, so the trees get shuffled and rebased a lot. 

Related to the preparatory trees are the "lead-in" trees run by sub-subsystem maintainers. The wireless networking and Bluetooth trees were held up as an example. These trees are "mostly OK," Stephen said, though some of them run on their own schedules. The Bluetooth tree, he noted, acquired a bunch of new work during the merge window, a practice [![\[Stephen Rothwell\]](https://static.lwn.net/images/conf/2014/ks/StephenRothwell-sm.jpg)](<https://lwn.net/Articles/608940/>) that Stephen discourages. If new patches flow into linux-next during that time, it complicates the task of figuring out what hasn't been merged into the mainline yet. 

A significant change to linux-next is the addition of build-test coverage for the ARM architecture; until recently, only x86 and PowerPC were tested. Stephen expressed some amazement at how often an "allmodconfig" build test fails; sometimes, developers are putting code into linux-next that clearly has never been compiled. That runs counter to the rule that all linux-next code should be ready for the mainline. While an occasional build failure for an architecture like PowerPC might be understandable, he said, there really is no excuse for failing to build on x86; that is a fundamental test that every developer should be running. Linus added that he gets grumpy when that kind of code is directed his way as well. 

Josh Triplett suggested that, perhaps, all code should have to go through a public build test before being accepted into linux-next. The problem there is that linux-next _is_ that build test, for all practical purposes. Additionally, Fengguang Wu's automatic test system should be finding build problems. We don't, Stephen said, need a more formal process; we just need developers to pay a bit more attention. 

Rafael Wysocki said that, while he appreciated the merge conflict resolution work that Stephen does, he would rather fix conflicts in his trees himself. The problem, Stephen said, is that when he hits a conflict in the process of pulling trees into linux-next, he does not have the time to contact the appropriate maintainer and get a fix from them. It's faster and easier just to deal with it himself. And, in any case, the conflict may come from somewhere else anyway. In general, Stephen said, he gets four or five conflicts on a normal day, which is not that hard to deal with. Just before the merge window, when the tree is big and everybody is trying to get their code in place, things get much worse, though. That can lead to twelve-hour days, leading Ted Ts'o to wonder if the job isn't getting too big for one person to do. 

There is, Stephen said, still too much rebasing of trees that feed into linux-next. Sometimes it's unavoidable, but a tree should normally be in a finished state before being put into linux-next. 

A developer asked: what should be done about a patch that has found its way into linux-next and which turns out to be bad? Should the tree containing that patch be rewritten, causing the bad patch to disappear from the change history entirely? Or is it better to simply revert the patch at the head of the tree? Linus answered that dropping the patch (and rewriting the tree) is, essentially, a rebasing of the tree — and he tends not to respond well when asked to pull a just-rebased tree. There can be value in doing that; it makes the history cleaner. But it should not be done just before sending a pull request; at that point the patch should just be reverted. Having the patch and the revert in the history is embarrassing, he said, but in the end it's fine. Unless, he added, your tree has more reverts than commits, in which case there may be a problem. 

While things can always be improved, it would appear from Stephen's report that the linux-next tree is working reasonably well. In the years that it has been operating, it has become a vital part of the kernel development process, helping the merge window (and the whole development cycle) to go more smoothly. 

**Next** : [Kernel tinification](<https://lwn.net/Articles/608945/>).
