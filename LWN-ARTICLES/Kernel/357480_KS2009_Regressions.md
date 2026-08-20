---
title: "KS2009: Regressions"
url: https://lwn.net/Articles/357480/
date: "October 19, 2009"
category: "Development model-Regressions"
author: "By Jonathan Corbet October 19, 2009 LWN's 2009 Kernel Summit coverage"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 19, 2009

* * *

LWN's [2009 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2009/>)

Regressions are the worst sort of bug. As long as working systems continue to work, we can be reasonably confident that the kernel is getting better over time. But if we break working systems, the situation is far more questionable. For this reason, it is important to avoid introducing regressions into the kernel. When they occur, they must be recognized as such, tracked, and fixed as quickly as possible. The kernel's regression tracker is Rafael Wysocki; he presented some information on how we have been doing over the last year. 

Various plots showing the number of regressions reported and the number fixed against the age of each kernel were presented. These plots can be fitted to an exponential distribution which is very similar to that seen for phenomena like radioactive decay. Looking at things that way, Rafael says, regressions have a half life of about 17 days, and a mean lifetime of 24 days. 

There is an important qualification to bear in mind here, though: these plots only show regressions which have been fixed. Over the year or so surveyed, 858 regressions were reported, but only 738 of those were reported to be fixed. So, says Rafael, the full picture has to include a second type of regression which is harder to find and to fix. 

The 2.6.30 kernel, it seems, has a steeper regression curve than its predecessors; it is not flattening out in the same way. But the number of regressions reported is fewer for this point in the cycle. A fair amount of time was spent trying to figure out just what that means. Perhaps this kernel is simply better, with fewer bugs to report in the first place. Or perhaps there are fewer testers. Rafael believes that the number of testers is roughly constant, as is the rate at which they can find bugs. But, as the kernel grows, there is more code which will surely have bugs in it. So Rafael fears that more regressions are slipping through the cracks. 

This worry was echoed by others, some of whom noted that Rafael's regression list contains only a small subset of the problems. Distribution bug trackers tend to fill with many more issues. There doesn't seem to be any easy way to propagate distribution bug data upward, though; developers for distributions which ship relatively young kernels tend to be working flat-out already. 

Ted Ts'o noted, though, that he feels much safer running -rc1 kernels now than in the past. They really seem to be getting more stable. 

Arjan van de Ven put up some data from [Kerneloops.org](<http://kerneloops.org/>) suggesting that the number of bugs per kernel release remains roughly constant. As always, most users are affected by a relatively small number of bugs. In an aside, some questioned whether the posting of certain types of crashes - null pointer dereferences in particular - could be a security concern, but Alan Cox responded that null-pointer problems are being reported on the mailing lists all the time and few people are doing anything about them. A listing on kerneloops.org seems unlikely to worsen the situation. 

So how can we make things better? Rafael suggested that, in the current development model, the opening of the merge window tends to distract developers just when the just-released kernel is starting to get more users. That, in turn, can keep them from looking at regression reports. So he suggested that it might make sense to delay the opening of the merge window for one week after a major kernel release. Linus did not like the idea, though, saying that it would just drag out releases without helping things much. Rather than pay attention to regression reports, developers would just continue to bash out new stuff for the delayed merge window, creating even more regressions the next time around. 

What would really help, it was agreed, is more testers - hardly a new or novel conclusion. Perhaps if more people would run linux-next, more bugs would be found (and fixed) early. Linux-next is hard to test, though, and hard to debug. It changes radically from one day to the next, and it is not bisectable. Rafael thinks that there is little benefit to users to testing it, since any bugs found there are relatively unlikely to be fixed. Andrew Morton thought that developers should be testing their code in linux-next as a way of finding bugs caused by interactions with other changes. That is a hard sell, though; working with linux-next can force developers to contend with bugs from completely unrelated changes. Alan Cox noted that linux-next is often more reliable than the -rc1 releases. 

There was some talk about how long it can take for regression fixes to get into the mainline. Often these fixes will set in maintainer trees for some time. Might there be a way to get them in more quickly? As it turns out, a number of subsystem maintainers feel that these fixes should age for a while in a testing tree, lest they introduce other problems. So that situation is not likely to change much. 

[Next: The future of perf events](<https://lwn.net/Articles/357481/>)
