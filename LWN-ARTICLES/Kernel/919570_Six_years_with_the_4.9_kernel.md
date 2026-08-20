---
title: Six years with the 4.9 kernel
url: https://lwn.net/Articles/919570/
date: "January 12, 2023"
category: "Releases-4.9"
author: "By Jonathan Corbet January 12, 2023"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 12, 2023

The release of the [4.9.337](<https://lwn.net/ml/linux-kernel/167309164122758@kroah.com/>) stable kernel update on January 7 marked the end of an era: after just over six years of maintenance, the 4.9.x series will receive no more updates. This kernel saw a lot of change after Linus Torvalds made the "final" release and left the building; it's time for a look at the "stable" portion of this kernel's life to see what can be learned. 

The development cycle that led up to the 4.9 release saw the addition of 16,214 non-merge changesets contributed by 1,719 developers (a record at the time) working for (at least) 228 companies. In the six years between 4.9 and 4.9.337, instead, it gained 23,391 non-merge changesets from 4,037 developers working for at least 503 companies. The 4.9.337 release contains 114,000 more lines of code than 4.9 did. Rather than being the end of a kernel's development life, the final release from Torvalds is really just the beginning of a new and longer phase — at least, for long-term-support kernels. 

#### Contributors

The top contributors of fixes to 4.9.x were: 

> Top bug-fix contributors to 4.9.x  
> ---  
> Developer| Changesets| Pct  
> Greg Kroah-Hartman | 470| 2.0%  
> Eric Dumazet | 395| 1.7%  
> Johan Hovold | 356| 1.5%  
> Dan Carpenter | 326| 1.4%  
> Takashi Iwai | 295| 1.3%  
> Arnd Bergmann | 286| 1.2%  
> Thomas Gleixner | 196| 0.8%  
> Jason A. Donenfeld | 171| 0.7%  
> Eric Biggers | 159| 0.7%  
> Colin Ian King | 138| 0.6%  
> Christophe JAILLET | 134| 0.6%  
> Nathan Chancellor | 125| 0.5%  
> Hans de Goede | 120| 0.5%  
> Geert Uytterhoeven | 117| 0.5%  
> Xin Long | 113| 0.5%  
> Yang Yingliang | 108| 0.5%  
> Jan Kara | 102| 0.4%  
> Randy Dunlap | 101| 0.4%  
> Linus Torvalds| 92| 0.4%  
> Johannes Berg | 92| 0.4%  
> Peter Zijlstra | 91| 0.4%  
> Al Viro | 90| 0.4%  
> Florian Fainelli | 89| 0.4%  
> Theodore Ts'o | 88| 0.4%  
  
While Greg Kroah-Hartman shows as the top contributor of changesets, it is worth remembering that 337 of them are simply setting the version number for each stable release. His appearance there is thus an artifact of how the stable kernels are produced — not that he doesn't play a major role in this process, of course, as will be seen below. 

The most active employers of contributors to 4.9.x were: 

> Employers supporting 4.9.x fixes  
> ---  
> Company| Changesets| Pct  
> (Unknown)| 2177| 9.3%  
> (None)| 2149| 9.2%  
> Google| 1940| 8.3%  
> Red Hat| 1911| 8.2%  
> Intel| 1553| 6.6%  
> SUSE| 1181| 5.0%  
> Huawei Technologies| 1050| 4.5%  
> IBM| 834| 3.6%  
> (Consultant)| 767| 3.3%  
> Linux Foundation| 697| 3.0%  
> Linaro| 625| 2.7%  
> Arm| 434| 1.9%  
> Oracle| 387| 1.7%  
> Mellanox| 314| 1.3%  
> Samsung| 286| 1.2%  
> Broadcom| 260| 1.1%  
> Linutronix| 234| 1.0%  
> Facebook| 226| 1.0%  
> Renesas Electronics| 201| 0.9%  
> NXP Semiconductors| 196| 0.8%  
  
It can be interesting to compare these numbers to [the statistics for the 4.9 release](<https://lwn.net/Articles/708266/>). There are many of the same names there, but the ordering is different. The biggest contributors of work for a mainline release may not be the biggest contributors of fixes after that release is made. 

#### Backports

The stable rules require that changes appear in the mainline before being added to a stable update, so most (or all) of the patches counted above were written for the mainline. Backporting them to 4.9 is a different level of work on top of that. This task can be as simple as applying a patch unmodified to a different tree, or as complex as rewriting it altogether. Either way, there is clearly a lot of work involved in backporting over 23,000 patches to a different kernel. 

One way to try to separate out that work was suggested by Srivatsa S. Bhat. A developer who backports a patch to an older kernel is essentially resubmitting it, and so must add a Signed-off-by tag to the patch changelog. Each patch in the stable kernel also contains the commit ID of the original in the mainline. Using that information, one can look at each stable patch and identify any Signed-off-by tags that were added since that patch was merged into the mainline. Those additional signoffs should indicate who backported each one. 

In the 4.9.x series, 21,495 of the commits have added Signed-off-by tags. The remaining ones will include the above-mentioned version-number changes, patches that should have gotten an additional tag but didn't, and (most probably) patches that were backported by their original author. The result is thus a picture that is not perfect, but which is clear enough: 

> Top 4.9.x backporters  
> ---  
> Developer| Changesets| Pct  
> Greg Kroah-Hartman| 15135| 70.41%  
> Sasha Levin| 9208| 42.84%  
> Ben Hutchings| 310| 1.44%  
> David Woodhouse| 142| 0.66%  
> Amit Pundir| 90| 0.42%  
> Sudip Mukherjee| 83| 0.39%  
> Jason A. Donenfeld| 73| 0.34%  
> Mark Rutland| 71| 0.33%  
> Lee Jones| 52| 0.24%  
> Nathan Chancellor| 44| 0.20%  
> Florian Fainelli| 42| 0.20%  
> David A. Long| 40| 0.19%  
> Nick Desaulniers| 36| 0.17%  
> Alex Shi| 27| 0.13%  
> Thomas Gleixner| 24| 0.11%  
> James Morse| 24| 0.11%  
> Giuliano Procida| 24| 0.11%  
> Nobuhiro Iwamatsu| 23| 0.11%  
> Thadeu Lima de Souza Cascardo| 22| 0.10%  
> Arnd Bergmann| 15| 0.07%  
  
The bulk of the backporting work is clearly being done by the two stable-kernel maintainers: Kroah-Hartman and Sasha Levin. In some cases, they have both added signoffs to the same patch, causing the percentages to add up to more than 100%. The work done by everybody else pales by comparison — especially if one only looks at the patch counts. Often, though, the reason for a developer other than the stable-kernel maintainers to backport a patch is that the backport is not trivial. So, while the other developers backported far fewer patches, many of those patches almost certainly required a lot more work. 

#### Bug reports

In theory, almost every patch in the stable series is a bug fix, implying that somebody must have found and reported a bug. As it happens, only 4,236 of the commits in the 4.9.x series include a Reported-by tag — only 18% of the total. So most of the problems being fixed are either coming to light in some other way, or the report tags are not being included. For the patches that _did_ include such tags, the results look like: 

> Top bug reporters in 4.9.x  
> ---  
> Reporter| Reports| Pct  
> Syzbot| 901| 18.8%  
> Hulk Robot | 181| 3.8%  
> kernel test robot | 156| 3.2%  
> Dmitry Vyukov | 100| 2.1%  
> Andrey Konovalov | 80| 1.7%  
> Dan Carpenter | 79| 1.6%  
> Jann Horn | 34| 0.7%  
> Guenter Roeck | 29| 0.6%  
> Jianlin Shi | 27| 0.6%  
> Ben Hutchings | 26| 0.5%  
> Fengguang Wu | 26| 0.5%  
> Al Viro | 21| 0.4%  
> Arnd Bergmann | 19| 0.4%  
> Lars-Peter Clausen | 19| 0.4%  
> Xu, Wen | 19| 0.4%  
> Eric Biggers | 18| 0.4%  
> Igor Zhbanov | 18| 0.4%  
> TOTE Robot | 18| 0.4%  
> Tetsuo Handa | 17| 0.4%  
> Linus Torvalds| 16| 0.3%  
  
Bug reporting is clearly widely distributed, with the top three reporters (all robots) accounting for just over 25% of the total. Even so, it is clear that the bug-hunting robots are finding a lot of problems, hopefully before our users do. 

#### Bug introduction

Another thing one can look at is the source of the bugs that were fixed in 4.9.x. Some work mapping Fixes tags in 4.9.x commits to the original commits can shine a light on when bugs were introduced; the result is a plot that looks like this: 

> ![\[4.9.x fixes\]](https://static.lwn.net/images/2023/4.9-fixes-1.png)

The 4.9 and 4.8 releases are, unsurprisingly, the source of many of the bugs fixed in the stable updates, with nearly 700 coming from each. After that comes the usual long tail, including every release ever made since the Git era began at 2.6.12. Every pre-4.10 release in the Git history is represented here; the least-fixed release is 2.6.17, which was released in 2006, with "only" 22 fixes. 

That plot is not the whole story, though. Each of the 4.9.28, 4.9.34, 4.9.51, 4.9.75, 4.9.77, 4.9.78, 4.9.79, 4.9.94, 4.9.102, 4.9.187, 4.9.194, 4.9.195, 4.9.198, 4.9.207, 4.9.214, 4.9.219, 4.9.228, 4.9.253, 4.9.258, 4.9.259, 4.9.261, 4.9.265, 4.9.298, and 4.9.299 releases included a commit that was identified by a later Fixes tag; 4.9.81 and 4.9.218 had two, and 4.9.310 had three. Each of those, clearly, indicates a regression that was introduced into the stable kernel and later fixed. But even that is not the full picture; consider this: 

> ![\[post-4.9.x fixes\]](https://static.lwn.net/images/2023/4.9-fixes-2.png)

Every release made _after_ 4.9 also introduced bugs that had to be fixed in the stable updates — over 1,500 fixes in all. That is a lot of buggy commits to have introduced into a "stable" kernel. One should also not take the wrong message from the lower counts for more recent kernel releases. It _might_ be possible that our releases are getting less buggy, but a more plausible explanation is that the empty space in the upper-right half of that plot just represents bugs that have not yet been found and fixed. 

The 4.9 stable series was, thus, not perfect — not that anybody ever claimed that it was. It was, however, good enough to be the core of many deployed systems, including an unimaginable number of Android devices. The 4.9 kernel series is a testament to what the development community can accomplish when it sets its mind to it. It was a base that many users could rely on, and has well earned its retirement.
