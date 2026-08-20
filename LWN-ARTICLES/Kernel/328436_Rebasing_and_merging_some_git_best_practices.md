---
title: "Rebasing and merging: some git best practices"
url: https://lwn.net/Articles/328436/
date: "April 14, 2009"
category: "Development tools-Git; Git"
author: "By Jonathan Corbet April 14, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
April 14, 2009

In a typical development cycle, Linus Torvalds [pulls patches from over 100 git trees](<http://lwn.net/Articles/318699/>) into the mainline repository. While this is going on, it's not unusual for him to complain about how some of those trees are managed; most of the gripes have to do with excessive use of rebasing and merging operations. In a recent discussion on the dri-devel list, Linus [clarified his rules](<https://lwn.net/Articles/328438/>) somewhat on subsystem tree management. Your editor, on the theory that there might be a developer or two out there who does not read dri-devel, thought that it might be good to expose those rules more widely. 

The git "rebase" operation takes a set of patches applied to one tree and reworks them to apply to a different tree. If a developer has written some patches against 2.6.29, he or she can use "git rebase" to turn them into patches against 2.6.30-rc1 instead. With git, rebasing can also be used to make edits to the commit history. If something needs to be fixed in a patch which was made some time ago, the developer can (1) remove the more recent patches from the tree, (2) make the needed changes, and (3) rebase the removed patches back onto the fixed patch. This technique can be used to silently disappear an embarrassing bug from the history, improve patch changelogs, fix a patch conflict against somebody else's tree, and more. It's something that git-based developers simply end up doing occasionally. 

There are a couple of problems associated with rebasing, though. One of those is that it changes the commit history. Whenever a series of commits is rebased, anybody who was working with the old history is left out in the cold. If a heavily-used tree is rebased, all developers depending on that tree are forced to scramble to readjust to the new reality. The other problem is that rebased patches are changed patches; any testing that they saw may no longer be applicable. That is why Linus tends to grumble hard at trees which have obviously been rebased just prior to the sending of a pull request. The changes in those trees probably worked before the rebase, but the post-rebase changes have not been tested and may not work as well. 

Rebasing is clearly a useful technique, though. Linus does not tell developers not to use it; in fact, he encourages it sometimes. The key rule that was passed down is this: Thou Shalt Not Rebase Trees With History Visible To Others. If a developer has pulled in somebody else's tree, the resulting tree can no longer be rebased, since that would break the second developer's history. Similarly, once a tree has been exported such that others may be making use of it, it can no longer be rebased. 

On the other hand, private history can be rebased at will - and it probably should be. If a patch is seen to introduce a bug, it's best to fix it at the source rather than reverting it or adding a second, fixup patch; the result is a cleaner history which is less likely to create problems for people trying to bisect unrelated bugs. Your editor has found that rebasing is often needed to add tags ("Acked-by," for example) to patches which have been circulated for review. When one is creating a set of patches for the mainline kernel, one is really creating an entire history, not just the end result. Making that history clean and readable is to everybody's benefit. 

The associated rule that goes with this, though, is that trees which are subject to rebasing should not be exposed to the world: 

This means: if you're still in the "git rebase" phase, you don't push it out. If it's not ready, you send patches around, or use private git trees (just as a "patch series replacement") that you don't tell the public at large about. 

So, in other words, trees which might be rebased should be kept private. They should also not have other developers' trees pulled into them. 

It's worth noting that Linus very much practices what he preaches on this front. The mainline git repository accepts 10,000 or so changesets every development cycle, but it is never rebased. And that is a good thing: rebasing the mainline would cause massive pain throughout the development community. 

Merging is the other place where subsystem maintainers can run afoul of the Chief Penguin. A "merge" in git is similar to a merge in most other source code management systems; it joins two (or more) independent lines of development into the current branch. Git merges differ, though, in that they can have more than two incoming branches; Ingo Molnar is famous for his use of "octopus merges" joining vast numbers of branches in a single operation. In almost all cases, performing a merge adds a special commit to the repository indicating that the merge has been done and noting which files, if any, had conflicts. 

Merges go both ways. When Linus pulls a subsystem tree into the mainline, the result is a merge. But it is also common for developers to perform merges in the other direction; they will pull the mainline (or some higher-level subsystem tree) into a branch containing a local line of development. It is natural to want to develop code against the current state of the art; it gives confidence that the end result will work with everybody else's changes and minimizes the chances of an ugly merge conflict later on. 

But excessive pulling from the mainline (as evidenced by the merge commits which result) tends to irritate Linus. As he put it: 

But if I see a lot of "Merge branch 'linus'" in your logs, I'm not going to pull from you, because your tree has obviously had random crap in it that shouldn't be there. You also lose a lot of testability, since now all your tests are going to be about all my random code. 

As anybody who has worked with tip-of-the-repository kernels knows, the state of the mainline at any random point can be, well, random. So frequent pulling of the mainline into a development branch will add a certain amount of randomness to that branch; this randomness is not particularly helpful for somebody who is trying to get a feature working. It also increases the chances that another developer who ends up in the middle of the series while running a bisect operation will encounter unrelated bugs. So Linus would rather that developers not pull down from upstream trees: 

And, in fact, preferably you don't pull my tree at ALL, since nothing in my tree should be relevant to the development work _you_ do. Sometimes you have to (in order to solve some particularly nasty dependency issue), but it should be a very rare and special thing, and you should think very hard about it. 

The reality of the situation tends not to be so strict, though. An occasional merge to stay on top of what's happening elsewhere can make sense. What Linus suggests, though, is that the merges happen at specific release points. So pulling the tip of the mainline tree into a development tree probably does not make sense, but there might be an argument for pulling in 2.6.29 or 2.6.30-rc1. Doing things this way allows development to be based on a (hopefully) relatively stable point where the issues are known. 

The temptation to merge in the mainline during development can be hard to resist; one likes to know whether one's work is even remotely relevant to the current state of the code. Fortunately, git makes it really easy to create throwaway branches and test out merges and integration there. Once it's clear that things work, the test branch can be deleted and the (unmerged) development branch sent upstream. 

Similar rules apply to the merging of downstream code. The receiving repository should be in a reasonably well defined and stable state; typically developers maintain a "for upstream" branch for this kind of merge. And the downstream code should be "ready": it should be at some sort of release point and not in a random state of development. 

Of course, these rules [are not absolute](<https://lwn.net/Articles/328478/>): 

Git does allow people to do many different things, and solve problems different ways. I just want all the regular workflows to be "good practice", but then if you have to occasionally break the rules to solve some odd problem, go ahead and break the rules (and tell people why you did it that way this time). 

Linus first started playing with BitKeeper in February, 2002, so the kernel community now has seven years worth of experience with distributed version control. But the truth of the matter is that we are still figuring out the best way to use this particular tool. This is a process which is likely to continue for some time yet. As other large projects move toward using tools like git, they may want to look hard at the processes and rules which have been developed in the kernel community; they might just be able to shorten their own learning experience.
