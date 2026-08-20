---
title: "Revisiting stable-kernel regressions"
url: https://lwn.net/Articles/812231/
date: "February 13, 2020"
category: "Development model-Stable tree"
author: "By Jonathan Corbet February 13, 2020"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 13, 2020

Stable-kernel updates are, unsurprisingly, supposed to be stable; that is why the first of the [rules for stable-kernel patches](<https://www.kernel.org/doc/html/latest/process/stable-kernel-rules.html>) requires them to be ""obviously correct and tested"". Even so, for nearly as long as the kernel community has been producing stable update releases, said community has also been complaining about regressions that make their way into those releases. Back in 2016, LWN [did some analysis](<https://lwn.net/Articles/692866/>) that showed the presence of regressions in stable releases, though at a rate that many saw as being low enough. Since then, the volume of patches showing up in stable releases has grown considerably, so perhaps the time has come to see what the situation with regressions is with current stable kernels. 

As an example of the number of patches going into the stable kernel updates, consider that, as of 4.9.213, 15,648 patches have been added to the original 4.9 release — that is an entire development cycle worth of patches added to a "stable" kernel. Reviewing all of those to see whether each contains a regression is not practical, even for the maintainers of the stable updates. But there is an automated way to get a sense for how many of those stable-update patches bring regressions with them. 

The convention in the kernel community is to add a `Fixes` tag to any patch fixing a bug introduced by another patch; that tag includes the commit ID for the original, buggy patch. Since stable kernel releases are supposed to be limited to fixes, one would expect that almost every patch would carry such a tag. In the real world, about 40-60% of the commits to a stable series carry `Fixes` tags; the proportion appears to be increasing over time as the discipline of adding those tags improves. 

It is a relatively straightforward task (for a computer) to look at the `Fixes` tag(s) in any patch containing them, extract the commit IDs of the buggy patches, and see if those patches, too, were added in a stable update. If so, it is possible to conclude that the original patch was buggy and caused a regression in need of fixing. There are, naturally, some complications, including the fact that stable-kernel commits have different IDs than those used in the mainline (where all fixes are supposed to appear first); associating fixes with commits requires creating a mapping between the two. Outright reverts of buggy patches tend not to have `Fixes` tags, so they must be caught separately. And so on. The end result will necessarily contain some noise, but there is a useful signal there as well. 

For the curious, this analysis was done with the `stablefixes` tool, part of the `gitdm` collection of repository data-mining hacks. It can be cloned from `git://git.lwn.net/gitdm.git`. 

Back in 2016, your editor came up with a regression rate of at least 2% for the longer-term stable kernels that were maintained at that time. The 4.4 series, which had 1,712 commits then, showed a regression rate of at least 2.3%. Since then, the number of commits has grown considerably — to 14,211 in 4.4.213 — as a result of better discipline and the use of automated tools (including [a machine-learning system](<https://lwn.net/Articles/764647/>)) to select fixes that were not explicitly earmarked for stable backporting. Your editor fixed up his script, ported it to Python 3, and reran the analysis for the currently supported stable kernels; the results look like this. 

> Series| Commits| Tags| Fixes | Reverts|   
> ---|---|---|---|---|---  
> 5.4.18 | 2,423 | 1,482 | 61% | 74 | 29 | [Details](<https://lwn.net/Articles/812232/>)  
> 4.19.102 | 11,758 | 5,647 | 48% | 588 | 100 | [Details](<https://lwn.net/Articles/812233/>)  
> 4.14.170 | 15,527 | 6,727 | 43% | 985 | 134 | [Details](<https://lwn.net/Articles/812234/>)  
> 4.9.213 | 15,647 | 6,286 | 40% | 951 | 139 | [Details](<https://lwn.net/Articles/812235/>)  
> 4.4.213 | 14,210 | 5,110 | 36% | 834 | 124 | [Details](<https://lwn.net/Articles/812236/>)  
  
In the above table, **Series** identifies the stable kernel that was looked at. **Commits** is the number of commits in that series, while **Tags** is the number and percentage of those commits with a `Fixes` tag. The count under **Fixes** is the number of commits in that series that are explicitly fixing another commit applied to that series. **Reverts** is the number of those fixes that were outright reverts; a famous person might once have said that reversion is the sincerest form of patch criticism. Hit the "Details" link for a list of the fixes found for each series. 

Looking at those numbers would suggest that, for example, 3% of the commits in 5.4.18 are fixing other commits, so the bad commit rate would be a minimum of 3%. The situation is not actually that simple, though, for a few reasons. One of those is that a surprising number of the regression fixes appear in the same stable release as the commits they are fixing. In a case like that, while the first commit can indeed be said to have introduced a regression, no stable release actually contained the regression and no user will have ever run into it. Counting those is not entirely fair. If one subtracts out the same-release fixes, the results look like this: 

> Series| Fixes| Same  
> release| Visible  
> regressions  
> ---|---|---|---  
> 5.4.18 | 74 | 29 | 45  
> 4.19.102 | 588 | 176 | 412  
> 4.14.170 | 985 | 253 | 732  
> 4.9.213 | 951 | 229 | 722  
> 4.4.213 | 834 | 232 | 602  
  
Another question to keep in mind is what to do with all those commits without `Fixes` tags. Many of them are certainly fixes for bugs introduced in other patches, but nobody went to the trouble of figuring out how the bugs happened. If the numbers in the table above are taken as the total count of regressions in a stable series, that implies that none of the commits without `Fixes` tags are fixing regressions, which will surely lead to undercounting regression fixes overall. On the other hand, if one assumes that the untagged commits contain regression fixes in the same proportion as the tagged ones, the result could well be a count that is too high. 

Perhaps the best thing that can be done is to look at both numbers, with a reasonable certainty that the truth lies somewhere between them: 

> Series| Visible  
> regressions | Regression rate  
> ---|---|---  
> Low| High  
> 5.4.18 | 45 | 1.9% | 3.0%  
> 4.19.102 | 412 | 3.5% | 7.3%  
> 4.14.170 | 732 | 4.7% | 10.9%  
> 4.9.213 | 722 | 4.6% | 11.5%  
> 4.4.213 | 602 | 4.2% | 11.8%  
  
So that is about as good as the numbers are going to get, though there are still some oddball issues. Consider the case of [mainline commit 4abb951b73ff](<https://git.kernel.org/linus/4abb951b73ff>) ("ACPICA: AML interpreter: add region addresses in global list during initialization"). This commit included a "`Cc: stable@vger.kernel.org`" tag, so it was duly included (as [commit 22083c028d0b](<https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit?id=22083c028d0b3ee419232d25ce90367e5b25df8f>)) in the 4.19.2 release. It was then [reverted](<https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit?id=8ef305fbc50d93cc7e2f594abcf9546f3afbd435>) in 4.19.3, with the complaint that it didn't actually fix a bug but did cause regressions. This same change [returned](<https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit?id=87403e35bc56>) in 4.19.6 after an explicit [request](<https://lwn.net/ml/linux-kernel/20181214174220.GA30170%40kroah.com/>). Then, two commits followed in 4.19.35: commit [d4b4aeea5506](<https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit?id=d4b4aeea5506>) addressed a related issue and the original upstream commit in a `Fixes` tag, while [f8053df634d4](<https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit?id=f8053df634d4>) claimed to _be_ the original upstream commit, which had already been applied. That last one looks like a fix for a partially done backport. How does one try to account for a series of changes like that? Honestly, one doesn't even try. 

So what can we conclude from all this repository digging? The regression rates seen in 2016 were quite a bit lower than what we are seeing now; that would suggest that the increasing volume of patches being applied to the stable trees is not just increasing the number of regressions, but also the rate of regressions. That is not a good sign. On the other hand, the amount of grumbling about stable regressions seems to have dropped recently. Perhaps that's just because people have gotten used to the situation. Or perhaps the worst problems, such as filesystem-destroying regressions, are no longer getting through, while the problems that do slip through are relatively minor. 

Newer kernels have a visibly lower regression rate than the older ones. There are two equally plausible explanations for that. Perhaps the process of selecting patches for stable backporting is getting better, and fewer regressions are being introduced than were before. Or perhaps those kernels just haven't been around for long enough for all of the regressions already introduced to be found and fixed yet. The 2016 article looked at 4.4.14, which had 39 regression fixes (19 fixed in the same release). 4.4.213 now contains 110 fixes for regressions introduced in 4.4.14 or earlier (still 19 fixed in the same release). So there is ample reason to believe that the regression rate in 5.4.18 is higher than indicated above. 

In any case, it seems clear that the push to get more and more fixes into the stable trees is unlikely to go away anytime soon. And perhaps that is a good thing; a stable tree with thousands of fixes and a few regressions may still be far more stable than one without all those patches. Even so, it would be good to keep an eye on the regression rate; if that is allowed to get too high, the result is likely to be users moving away from stable updates, which is definitely not the desired result.
