---
title: On the scalability of Linus
url: https://lwn.net/Articles/393694/
date: "July 2, 2010"
category: Development model
author: "By Jonathan Corbet July 2, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 2, 2010

The Linux kernel development process stands out in a number of ways; one of those is the fact that there is exactly one person who can commit code to the "official" repository. There are many maintainers looking after various subsystems, but every patch they merge must eventually be accepted by Linus Torvalds if it is to get into the mainline. Linus's unique role affects the process in a number of ways; for example, as this article is being written, Linus has just returned from a vacation which resulted in nothing going into the mainline for a couple of weeks. There are more serious concerns associated with the single-committer model, though, with scalability being near the top of the list. 

Some LWN readers certainly remember the 2.1.123 release in September, 1998\. That kernel failed to compile due to the incomplete merging (by Linus) of some frame buffer patches. A compilation failure in a development kernel does not seem like a major crisis, but this mistake served as a flash point for anger which had been growing in the development community for a while: patches were increasingly being dropped and people were getting tired of it. At the end of a long and unpleasant discussion, Linus [threw up his hands](<http://lkml.indiana.edu/hypermail/linux/kernel/9809.3/0850.html>) and walked away: 

Quite frankly, this particular discussion (and others before it) has just made me irritable, and is ADDING pressure. Instead, I'd suggest that if you have a complaint about how I handle patches, you think about what I end up having to deal with for five minutes. 

Go away, people. Or at least don't Cc me any more. I'm not interested, I'm taking a vacation, and I don't want to hear about it any more. In short, get the hell out of my mailbox. 

This was, of course, the famous "Linus burnout" episode of 1998. Everything stopped for a while until Linus rested a bit, came back, and started merging patches again. Things got kind of rough again in 2000, leading to Eric Raymond's somewhat sanctimonious [curse of the gifted lecture](<http://lwn.net/2000/0824/a/esr-sharing.php3>). In 2002, as the 2.5 series was getting going, frustration with dropped patches was, again, on the rise; Rob Landley's ["patch penguin" proposal](<http://lwn.net/2002/0131/a/patch-penguin.php3>) became the basis for yet another extended flame war on the dysfunctional nature of the kernel development process and the "Linus does not scale" problem. 

Shortly thereafter, things got a whole lot smoother. There is no doubt as to what changed: the adoption of BitKeeper - for all the flame wars that it inspired - made the kernel development process work. The change to Git improved things even more; it turns out that, given the right tools, Linus scales very well indeed. In 2010, he handles a volume of patches which would have been inconceivable back then and the process as a whole is humming along smoothly. 

Your editor, however, is concerned that there may be some clouds on the horizon; might there be another Linus scalability crunch point coming? In the 2.6.34 cycle, Linus established a policy of unpredictable merge window lengths - though that policy has been more talk than fact so far. For 2.6.35, quite a few developers saw the merge window end with no response to their pull requests; Linus simply decided to ignore them. The [blowup over the ARM architecture](<http://lwn.net/Articles/391372/>) was part of this, but quite a few other trees remained unpulled as well. We have not gone back to the bad old days where patches would simply disappear into the void, and perhaps Linus is just experimenting a bit with the development process to try to encourage different behavior from some maintainers. Still, silently ignoring pull requests does bring back a few memories from that time. 

#### Too many pulls?

A typical development cycle sees more than 10,000 changes merged into the mainline. Linus does not touch most of those patches directly, though; instead, he pulls them from trees managed by subsystem maintainers. How much attention is paid to specific pull requests is not entirely clear; he does look at each closely enough to ensure that it contains what the maintainer said would be there. Some pulls are obviously subjected to closer scrutiny, while others get by with a quick glance. Still, it's clear that every pull request and every patch will require a certain amount of attention and thought before being merged. 

The following table summarized mainline merging activity by Linus over the last ten kernel releases (the 2.6.35 line is through 2.6.35-rc3): 

> Release| Pulls | Patches  
> ---|---|---  
> MergeWin| Total| Direct| Total  
> 2.6.26| 159| 426 | 288| 1496  
> 2.6.27| 153| 436 | 339| 1413  
> 2.6.28| 150| 398 | 313| 954  
> 2.6.29| 129| 418 | 267| 896  
> 2.6.30| 145| 411 | 249| 618  
> 2.6.31| 187| 479 | 300| 788  
> 2.6.32| 185| 451 | 112| 789  
> 2.6.33| 176| 444 | 104| 605  
> 2.6.34| 118| 393 | 94| 581  
> 2.6.35| 160| 218 | 38| 405  
  
The two columns under "pulls" show the number of trees pulled during the merge window and during the development cycle as a whole. Note that it's possible that these numbers are too small, since "fast-forward" merges do not necessarily leave any traces in the git history. Linus does very few fast-forward merges, though, so the number of missed merges, if any, will be small. 

Linus still directly commits some patches into his repository. The bulk of those come from Andrew Morton, who does not use git to push patches to Linus. In the table above, the "total" column includes changes that went by way of Andrew, while the "direct" column only counts patches that Andrew did not handle. 

Some trends are obvious from this table: the number of patches going directly into the mainline has dropped significantly; almost everything goes through somebody else's tree first. What's left for Linus, at this point, is mostly release tagging, urgent fixes, and reverts. Andrew Morton remains the maintainer of last resort for much of the kernel, but, increasingly, changes are not going through his tree. Meanwhile, the number of pulls is staying roughly the same. It is interesting to think about why that might be. 

Perhaps there is no need for more pulls despite the increase in the number of subsystem trees over time. Or perhaps we're approaching the natural limit of how many subsystem pull requests one talented benevolent dictator can pay attention to without burning out. After all, it stands to reason that the number of pull requests handled by Linus cannot increase without bound; if the kernel community continues to grow, there must eventually be a scalability bottleneck there. The only real question is where it might be. 

[![\[2.6.35 merge paths\]](https://static.lwn.net/images/ns/kernel/trees-2.6.35-sm.png)](<https://lwn.net/Articles/393814/>) If there is a legitimate concern here, then it might be worth contemplating a response before things break down again. One obvious approach would be to change the fact that almost all trees are pulled directly into the mainline; see [this plot](<https://lwn.net/Articles/393814/>) to see just how flat the structure is for 2.6.35. Subsystem maintainers who have earned sufficient trust could possibly handle more lower-level pull requests and present a result to Linus that he can merge with relatively little worry. The networking subsystem already works this way; a number of trees feed into David Miller's networking tree before being sent upward. Meanwhile, other pressures have led to the opposite thing happening with the ARM architecture: there are now several subarchitecture trees which go straight to Linus. The number of ARM pulls seems to have been a clear motivation for Linus to shut things down during the 2.6.35 merge window. 

Another solution, of course, would be to empower others to push trees directly into the mainline. It's not clear that anybody is ready for such a radical change in the kernel development process, though. Ted Ts'o's [1998 warning](<http://lwn.net/1998/1008/a/tso-core.html>) to anybody wanting a "core team" model still bears reading nearly twelve years later. 

But if Linus is to retain his central position in Linux kernel development, the community as a whole needs to ensure that the process scales and does not overwhelm him. Doing more merges below him seems like an approach that could have potential, but the word your editor has heard is that Linus is resistant to too much coalescing of trees; he wants to look stuff over on its way into the mainline. Still, there must be places where this would work. Maybe we need an overall ARM architecture tree again, and perhaps there could be a place for a tree which would collect most driver patches. 

The Linux kernel and its development process have a much higher profile than they did even back in 2002. If the process were to choke again due to scalability problems at the top, the resulting mess would be played out in a very public way. While there is no danger of immediate trouble, we should not let the smoothness of the process over the last several years fool us into thinking that it cannot happen again. As with the code itself, it makes sense to think about the next level of scalability issues in the development process before they strike.
