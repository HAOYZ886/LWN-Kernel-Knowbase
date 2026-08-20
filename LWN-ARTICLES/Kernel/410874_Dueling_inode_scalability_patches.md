---
title: Dueling inode scalability patches
url: https://lwn.net/Articles/410874/
date: "October 20, 2010"
category: Scalability
author: "By Jonathan Corbet October 20, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 20, 2010

Nick Piggin's [VFS scalability patch set](<http://lwn.net/Articles/360199/>) has been under development for well over a year. Some pieces were merged for 2.6.36, but the more complicated parts were deferred because Nick thought they needed more work and more testing. Then things went quiet; Nick changed jobs and went on vacation, so little work was done for some time. Eventually it became clear that Nick was unlikely to get the scalability work into shape for a 2.6.37 merge. 

So Dave Chinner decided to jump in and work on these patches, and the code breaking up the inode lock in particular. His [first patch set](<http://lwn.net/Articles/407560/>) was posted in late September, with a number of revisions happening since. Dave worked on splitting the patch series into smaller, more reviewable chunks. He also took out some of the (to him) scarier changes. Subsequent revisions brought larger changes, to the point that [version 5](<https://lwn.net/Articles/410879/>) reads: 

None of the patches are unchanged, and several of them are new or completely rewritten, so any previous testing is completely invalidated. I have not tried to optimise locking by using trylock loops - anywhere that requires out-of-order locking drops locks and regains the locks needed for the next operation. This approach simplified the code and lead to several improvements in the patch series (e.g. moving inode->i_lock inside writeback_single_inode(), and the dispose_one_inode factoring) that would have gone unnoticed if I'd gone down the same trylock loop path that Nick used. 

According to Dave, this patch set helps with the scalability problems he has been seeing, and other reviewers seem to think that the patch set is starting to look fairly good. 

But then Nick returned. While he welcomed the new interest in scalability work, he did not take long to indicate that he was not pleased with the direction in which Dave had taken his patches. He has posted [a 35-part patch series](<http://lwn.net/Articles/410457/>) which he hopes to merge; the patch posting also details why he doesn't like Dave's alternative approach. The ensuing discussion has been a bit rough in spots, though it has remained mostly focused on the technical issues. 

What it has not done, though, is to come up with any sort of conclusion. There are two patch sets out there; both deal with the intersection of the virtual filesystem layer and the memory management code. Much of the contention seems to be over whether "VFS people" or "memory management people" should have the ultimate say in how things are done. Given the difficult nature of both patch sets and the imminent opening of the 2.6.37 merge window, it seems fairly safe to say that neither will be merged unless Linus makes an executive decision. Pushing back this code to 2.6.38 will provide an opportunity for the patches to be discussed at length, and, possibly, for the upcoming Kernel Summit to consider them as well.
