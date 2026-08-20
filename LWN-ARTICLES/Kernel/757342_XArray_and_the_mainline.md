---
title: XArray and the mainline
url: https://lwn.net/Articles/757342/
date: "June 13, 2018"
category: XArray
author: "By Jake Edge June 13, 2018 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
June 13, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

The [XArray data structure](<https://lwn.net/Articles/745073/>) was the topic of the final filesystem track session at the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). XArray is a new API for the kernel's [radix-tree](<https://lwn.net/Articles/175432/>) data structure; the session was led by Matthew Wilcox, who created XArray. When asked by Dave Chinner if the session was intended to be a live review of the patches, Wilcox admitted with a grin that it might be "the only way to get a review on this damn patch set". 

In fact, the session was about the status of the patch set and its progress toward the mainline. Andrew Morton has taken the first eight cleanup patches, Wilcox said, which is great because there was a lot of churn there. The next set has a lot of churn as well, mostly due to renaming. The 15 patches after that actually implement XArray and apply it to the page cache. Those could be buggy, but they pass the radix-tree tests so, if they are, more tests are needed, he said. 

[ ![\[Matthew Wilcox\]](https://static.lwn.net/images/2018/lsf-wilcox-sm.jpg) ](<https://lwn.net/Articles/757340/>)

Jeff Layton wondered if XArray should spend some time in the linux-next tree. Chinner said that it should be in linux-next at the time of the presentation (April 25) if it was meant to go into 4.18. Wilcox said that his code is based on linux-next and he would check to see if he could get it into that tree. It would seem that his plan has been delayed by a development cycle based on his [post of XArray patches](<https://lwn.net/Articles/757140/>) on June 11\. 

Ted Ts'o asked about merge conflicts between the XArray patches and XFS, ext4, and Btrfs in Linus Torvalds's tree. Wilcox said there were few, since most of those were handled in Morton's merge. Wilcox said that he has only converted the page cache to use XArray in the patches. 

David Howells noted that the radix-tree code is still available in parallel for now. Wilcox said that code using XArray needs to use its locking scheme as well or lockdep will complain. The patches add roughly two-thirds the number of lines they delete; the "diffstat is wrongish" because there is lots more documentation and the radix tree is still present. 

Chinner asked about performance numbers; Wilcox said that he had not done any measurements, but he would welcome anyone who wanted to. Chinner said he had done some performance testing a ways back and found the difference to be in the noise. 

There are more radix-tree users in the kernel, though; Chinner wondered whether those conversions would go through the maintainer trees or via some other mechanism. Wilcox said that there are 50 or 60 radix trees in the kernel; he intends to allow XArray to live for a cycle in the mainline then start submitting conversions of other radix trees to their maintainers. Anna Schumaker asked when he planned to get rid of the radix tree entirely. Wilcox said he had about a dozen users of the radix tree left to convert in his tree and those are "not scary ones"; he thinks the radix tree could be gone within six months.
