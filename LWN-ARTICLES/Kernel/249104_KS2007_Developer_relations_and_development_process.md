---
title: "KS2007: Developer relations and development process"
url: https://lwn.net/Articles/249104/
date: "September 10, 2007"
category: "Development model-Developer conduct"
author: "By Jonathan Corbet September 10, 2007 LWN.net Kernel Summit 2007 coverage"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 10, 2007

* * *

LWN.net [Kernel Summit 2007](<https://lwn.net/Articles/KernelSummit2007/>) coverage 

James Bottomley started off the session on developer relations by pointing at Greg Kroah-Hartman's [2.6.22 developers poster](<http://lwn.net/Articles/240418/>) which was hanging in the back of the room and stating that the 900+ developers listed there were not anywhere near enough. If Linux is to continue to grow and stay healthy, we need to bring in many more developers. There are, he says, a couple of ways of doing that. By simply increasing the user base we should increase our developer base as well; some small percentage of Linux users will eventually start to hack on the system. The other way is to stop discouraging developers and pushing them out of the community. The latter topic took up most of this session. 

Flaming and generally unpleasant behavior remain a problem in the kernel community. Whenever one developer flames another - for something trivial like whitespace violations or something more substantial - he sets an example for others. The original developer may feel justified in the flaming by being "right," but those who follow may be less right while being just as inflammatory. The result is flaming by people who have never considered sending in a kernel patch. We are, says James, attracting idiots to our community by our behavior. 

We are also making it harder to recruit developers. Every flame can only be discouraging for all of those potential developers who are lurking on the sidelines and thinking about joining in. Flaming creates an unwelcoming environment and works against our interests in a big way. 

There was a brief discussion of filtering linux-kernel posts by the number of patches the author has committed. Or at least marking posts with that information. It was noted that such a policy would make it hard for users to submit bug reports. But, more to the point, the application of technical measures to a social problem like this one is unlikely to help a whole lot. So this discussion did not go very far. 

What could help would be for the leaders in the kernel community, many of whom were at the summit, to show their leadership in their behavior on the lists. Try to encourage people doing good things, especially if those people have been hit by flames from other directions. Gently discourage people who are seen to be attacking others. And, in general, work toward a better environment on our mailing lists. 

Incendiary email can be especially problematic for developers from some parts of the world where people simply do not talk to each other that way, especially in professional settings. In some cultures, being flamed on a mailing list can lead to career problems. If we want to encourage worldwide participation - and we do - we need to avoid creating problems for people who are unused to this sort of discourse. 

There are a lot of things which can be done to help in this regard beyond the simple avoidance of flaming. Starting a patch review with a "thank you" and some encouraging comments can work wonders. It was noted that all of this has little to do with specific cultures - it's just basic politeness that will make the process work better for everybody. 

Andrew Morton then took the podium to lead a session on the development process in general. He had a number of issues which were relevant to him in particular. For example, when a subsystem tree which he pulls into -mm is merged into the mainline, he would like the maintainer to update the tree so that there will not be merge conflicts for him to deal with. 

At the 2006 kernel summit, it was agreed that subsystem maintainers would send out a note sometime around the end of the development cycle describing what they planned to merge once the next cycle opened up. Very few maintainers have been doing that. Andrew still thinks it is a good idea, and has asked that people give it a try. Since then, a number of these notices have been posted to the mailing lists. 

There was some discussion about how the process should work when there are very large, cross-subsystem patches to be merged. An example would be the upcoming i386/x86_64 merger. Andrew prefers to merge these patches right at the end of the merge window, perhaps after -rc1 has come out. That minimizes the number of conflicts with all of the other patches. 

In each development cycle, something like 20% of the patches go into the mainline directly from -mm rather than coming from a subsystem tree. Andrew says he prefers to merge those at the end of the merge window, after the subsystem trees have gone in. To facilitate this process, he asks that developers not merge their trees at the last minute. It is a real pain for him when he has only the last few hours of the merge window to work through his huge pile of patches. 

Linus was a bit less diplomatic about it; he is tired of getting merge requests late in the window. So, he says, he wants to have merge requests in his mailbox on the first day of the merge window. This should not be a problem, he says; the code should be ready to go when the window opens. He is also increasingly annoyed by developers who merge "inappropriate stuff" after the merge window closes. This stuff includes new features, but also things like code cleanups and such. If it is not a bug fix, it really needs to be merged before -rc1. 

Is the -mm tree too experimental? Andrew says that it really is just the merger of the various subsystem trees; most of what is in -mm is intended for the next merge window. So it really _should_ work. And there is, as far as he can tell, no particular part of -mm which creates an inordinate number of problems. Mel Gorman said that it generally takes a day or two for him to get a new -mm release going on his test machines, though. 

There were requests for a more real-time view into the current status of the -mm tree. For now, all that most people see is the announcement Andrew sends out when he releases a new tree - something which, he says, has not been happening often enough recently. Curious people were pointed at the mm-commits mailing list, which receives a message every time a patch is added to or dropped from -mm. Your editor can attest, however, that this list is a good way to get a _lot_ of mail. 

Should user-space programs be distributed with the kernel? In particular, Andrew is wondering about the various utilities which have currently found a home in the documentation directory. The consensus seems to be that these utilities should be merged into the util-linux distribution instead. 

There was some talk about the -stable tree. Andrew fears that we are not doing a good enough job of identifying patches which should go into -stable. So he encourages all maintainers to think about it whenever they accept patches into their tree. Chuck Ebbert has apparently been looking over patches for candidates and notifying maintainers when he think they should consider submitting patches to -stable, but he has, at times, gotten less than friendly responses back. Dave Miller suggests sitting on -stable candidates for a week before sending them in; often further issues come up which force a rework of the patch. The one exception, of course, would be patches for severe security problems. 

It was generally agreed that the -stable tree is useful; distributors are using it to create the kernels they ship. 

Finally, Andrew brought up the topic of cleanup patches. Anybody who watches the kernel patch stream knows that there is a steady flow of patches which adjust coding style, fix spelling errors, etc. He hates these patches, but he has been forced to admit that they do, in the end, make the kernel better. So he accepts them. Most of the developers would seem to agree. 

What people would like to see, perhaps, is a more targeted approach to cleanup patches. Rather than trying to fix a small problem across the entire tree, pick a file or two with problems and try to take care of everything there. It was also noted that there have been problems with cleanup patches; some of them create subtle bugs or are a non-optimal solution to the problem. That would appear to be the way things are, though: many of these patches are from new developers who are trying to get into the process. So they will require review and revision. But this is a path by which we bring new hackers into the fold, so the effort is worthwhile.
