---
title: "The core of the -stable debate"
url: https://lwn.net/Articles/863505/
date: "July 22, 2021"
category: "Development model-Stable tree"
author: "By Jonathan Corbet July 22, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 22, 2021

Disagreements over which patches should find their way into stable updates are not new — or uncommon. So when the topic came up again recently, there was little reason to expect anything but more of the same. And, for the most part, that is what ensued but, in this exchange, we were also able to see the core issue that drives these discussions. There are, in the end, two fundamentally different views of what the stable tree should be. 

The [5.13.2](<https://lwn.net/ml/linux-kernel/20210712060912.995381202@linuxfoundation.org/>) stable update was not a small one; it contained an even 800 patches. That is 5% of the total size of the mainline 5.13 development cycle, which was not small either. With the other stable kernels going out for consideration on the same day, there were over 2,000 stable-bound patches in need of review; that is a somewhat tall order for even a large community to handle in the two days that are allowed. Even so, Hugh Dickins was able to [raise an objection](<https://lwn.net/ml/linux-kernel/2b1b798e-8449-11e-e2a1-daf6a341409b@google.com/>) over the inclusion of several memory-management patches that had not been specifically marked for inclusion in the stable releases. Those patches, he thought, were not suitable for a stable kernel and should not have been selected. 

Stable-kernel maintainer Greg Kroah-Hartman [responded](<https://lwn.net/ml/linux-kernel/YO0zXVX9Bx9QZCTs@kroah.com/>) that the size of the update was due to maintainers holding onto fixes until the merge window opens. Once the -rc1 release comes out, those fixes all land in the stable updates, which are, as a result, huge. But it is clear that the amount of change going into the stable kernels has been growing over time. If one looks at the number of changes going into the first five updates for each release (enough updates to include the merge-window fixes), the result is: 

> Release | Updates | Changes  
> ---|---|---  
> First 5| Total  
> 4.19 | 198 | 754 | 19,682  
> 5.0 | 21 | 408 | 2,387  
> 5.1 | 18 | 353 | 1,747  
> 5.2 | 21 | 779 | 2,429  
> 5.3 | 18 | 561 | 2,178  
> 5.4 | 134 | 435 | 14,414  
> 5.5 | 19 | 653 | 2,516  
> 5.6 | 19 | 364 | 1,864  
> 5.7 | 19 | 581 | 2,984  
> 5.8 | 18 | 899 | 2,755  
> 5.9 | 16 | 1,244 | 2,339  
> 5.10 | 52 | 841 | 7,295  
> 5.11 | 22 | 948 | 3,588  
> 5.12 | 19 | 1,446 | 3,843  
> 5.13 | 4 | 1,416 | 1,416  
  
While 5.13 has not yet reached five stable releases as of this writing, it seems safe to predict that it will collect more changes in its first five stable updates than any of its predecessors. The number of patches going into the stable updates is increasing in general, at a rate that would seem to exceed the growth in the rate at which changes are applied during mainline kernel development cycles. Long-term stable kernels now receive more patches during their "stable" period than during the development cycle leading up to their "final" release. In other words, the development cycle is not even close to being finished when Linus Torvalds applies a tag and leaves the building. 

Developers and maintainers can indicate that a mainline patch should be backported to the stable releases by including a "`CC: stable@vger.kernel.org`" tag, but that is not how most patches get there; of the 1,416 commits in 5.13.4, only 259 — 18% — contained such a tag. The stable maintainers have increasingly aggressively sought out mainline patches that look like fixes and put them into the stable releases. The `Fixes` tag found on many patches is used as a cue that a patch is a fix, but [machine learning is also being used](<https://lwn.net/Articles/764647/>) to select patches. The result is a lot of commits going into stable updates without having ever been explicitly marked for that treatment. 

This work has always been controversial, especially when regressions slip through. Regressions are inevitable regardless, but it is hard to imagine a way to add over 3,000 changes to a kernel during a three-month short-term stable cycle without a few of them being bad. The patches singled out by Dickins this time around are _not_ responsible for any regressions (that anybody knows about yet), but the memory-management developers are clearly worried about the possibility. 

To avoid any such outcome, memory-management maintainer Andrew Morton has [requested](<https://lwn.net/ml/linux-kernel/20210713182813.2fdd57075a732c229f901140@linux-foundation.org/>) that patches carrying his `Signed-off-by` tag not be included in stable updates in the absence of a specific request: ""At present this -stable promiscuity is overriding the (sometime carefully) considered decisions of the MM developers, and that's a bit scary"". Kroah-Hartman [asked](<https://lwn.net/ml/linux-kernel/YO7lZpqC4xrMPXQg@kroah.com/>) how the decision to mark a patch for stable backporting is made and, specifically, why a number of clear fixes were not selected; Morton [explained the thinking](<https://lwn.net/ml/linux-kernel/20210714140911.6c45f8f4a9b129ed36bb9d06@linux-foundation.org/>) this way: 

> Broadly speaking: end-user impact. If we don't have reports of the issue causing a user-visible problem and we don't expect such things to occur, don't backport. Why risk causing some regression when we cannot identify any benefit? 

Kroah-Hartman's [position](<https://lwn.net/ml/linux-kernel/YO6r1k7CIl16o61z@kroah.com/>), instead, is that, if a patch fixes a bug, it should be included in the stable updates: 

> But it really feels odd that you all take the time to add a "Hey, this fixes this specific commit!" tag in the changelog, yet you do not actually want to go and fix the kernels that have that commit in it. This is an odd signal to others that watch the changelogs for context clues. 

Sasha Levin (who also works on the stable updates) [added](<https://lwn.net/ml/linux-kernel/YO8DJkVzHFmPv6vz@sashalap/>) that a lot of important fixes are missed if only the explicitly tagged patches are backported into the stable kernels. Some of those fixes then find their way into distributor kernels via other paths, which doesn't seem ideal. 

In the end, this is what the disagreement comes down to: a difference of opinion on what is the best way to create stable updates that are truly stable and free of problems. 

  * Many developers see the stable updates as a carefully curated collection of hand-selected fixes, each of which has been extensively reviewed for importance and the lack of regression potential. These kernels should be safe to update to, since they should have a minimal chance of introducing problems not seen in their predecessors. This position tends to be taken by the developers of complex, core subsystems that have a high potential for subtle regressions. 

The memory-management subsystem is a classic example; there was also [a similar discussion with the XFS filesystem developers](<https://lwn.net/Articles/838819/>) in late 2020. Memory management requires predicting the future; as a result, the code is a large collection of complicated heuristics that have to work with a huge variety of workloads. It is not uncommon for an innocent-seeming change to create a performance regression for some private customer workload that won't surface until years have passed. Memory-management developers have learned that their lives run much more smoothly in the absence of such regressions, so they go out of their way to avoid making unnecessary changes to stable kernels. 

  * Others, including the stable maintainers, feel that the best kernel is to be had by including every fix that can be reasonably backported. Many bugs have user impacts — including security problems — that are not obvious to the developers when those bugs are being fixed; including all of the fixes will head off a lot of problems before they are discovered. 

Some distributors have taken this position and, as a result, are happy with how things are working; Justin Forbes [wrote](<https://lwn.net/ml/linux-kernel/CAFxkdAqE0vKCyr4qFjtKmn46rHn+RJsn7m_MX6jjbN6FZcDLMA@mail.gmail.com/>) that ""the current stable process has fixed more bugs than it has introduced"". The Android kernel is increasingly tied to the stable updates as a way of getting as many fixes as possible; based on this experience, Kroah-Hartman [said](<https://lwn.net/ml/linux-kernel/YO8GLupNQWpqsrT6@kroah.com/>): ""I have numbers to back up the other side, along with the security research showing that to ignore these stable releases puts systems at documented risk"". 

Which position is "correct" is not entirely clear, but there is no doubt about which position is "winning". As always in the Linux world, the people who are doing the work will decide how the work is to be done, and the stable maintainers have opted for the "promiscuous" approach. It would be interesting to see if there would be a user community for ultra-stable kernels maintained using the more restrictive approach, but it is doubtful that anybody has the time to create such a thing. 

There is always room for tweaking around the edges and opting out certain subsystems; this seems likely to happen with regard to memory management. More testing would also certainly help; the testing picture for stable releases has improved considerably over the years but could still get better. Ted Ts'o [suggested](<https://lwn.net/ml/linux-kernel/YO8EQZF4+iQ13QU%2F@mit.edu/>) that there could be a role for a "perfbot" system that looked for performance regressions in particular, if the resources could be found to create that sort of facility. Performance regressions are particularly difficult to test for, though; the resource requirements are large and it is nearly impossible to simulate every type of workload. 

In any case it seems that large stable updates will continue to be the rule going forward. With luck they will continue to become less regression-prone, but they will never be completely regression-free. So it is safe to predict that the debates over what should and should not go into stable updates will continue indefinitely.
