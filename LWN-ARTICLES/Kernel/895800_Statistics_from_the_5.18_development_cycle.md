---
title: Statistics from the 5.18 development cycle
url: https://lwn.net/Articles/895800/
date: "May 23, 2022"
category: "Releases-5.18"
author: "By Jonathan Corbet May 23, 2022"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 23, 2022

The 5.18 kernel was [released](<https://lwn.net/ml/linux-kernel/CAHk-=wjiqyoH6qntYvYTjR1F2L-pHtgX9esZMRS13iktCOJ1zA@mail.gmail.com/>) on May 22 after a nine-week development cycle. That can only mean that the time has come to look at some of the statistics behind this release, which was one of the busiest in a while. Read on for a look at the 5.18 kernel, where the code in this release came from, and how it found its way into the mainline. 

The 5.18 development cycle saw the addition of 14,954 non-merge changesets from 2,024 developers, 289 of whom made their first kernel contribution during this time. None of these numbers are records, though the number of developers came close to the maximum seen so far (2,062 for 5.13). This work resulted in the addition of 756,00 lines of code to the kernel. 

The top contributors to 5.18 were: 

> Most active 5.18 developers  
> ---  
> | By changesets  
> ---  
> Krzysztof Kozlowski | 214| 1.4%  
> Matthew Wilcox| 164| 1.1%  
> Christoph Hellwig | 154| 1.0%  
> Geert Uytterhoeven | 140| 0.9%  
> Ville Syrjälä | 135| 0.9%  
> Jonathan Cameron | 119| 0.8%  
> Andy Shevchenko | 118| 0.8%  
> Lorenzo Bianconi | 117| 0.8%  
> Vladimir Oltean | 111| 0.7%  
> Hans de Goede | 110| 0.7%  
> Martin Kaiser | 110| 0.7%  
> Colin Ian King | 104| 0.7%  
> Sean Christopherson | 100| 0.7%  
> Jakub Kicinski | 100| 0.7%  
> Christophe JAILLET | 89| 0.6%  
> Michael Straube | 87| 0.6%  
> Jani Nikula | 86| 0.6%  
> Trond Myklebust | 81| 0.5%  
> Eric Dumazet | 80| 0.5%  
> Christophe Leroy | 80| 0.5%  
> | By changed lines  
> ---  
> Leo Li | 227676| 19.4%  
> Qingqing Zhuo | 197757| 16.9%  
> Ian Rogers | 72008| 6.1%  
> Alan Kao | 15814| 1.3%  
> Ming Qian | 12176| 1.0%  
> Linus Walleij | 8881| 0.8%  
> Krzysztof Kozlowski | 8844| 0.8%  
> Dimitris Michailidis | 8791| 0.7%  
> Christoph Hellwig | 7165| 0.6%  
> Matt Roper | 7114| 0.6%  
> Jakub Kicinski | 7040| 0.6%  
> Jacob Keller | 6877| 0.6%  
> Geert Uytterhoeven | 6039| 0.5%  
> Ranjani Sridharan | 5768| 0.5%  
> Evan Quan | 5232| 0.4%  
> Guodong Liu | 4944| 0.4%  
> Mauro Carvalho Chehab | 4816| 0.4%  
> Vladimir Oltean | 4776| 0.4%  
> Brett Creeley | 4660| 0.4%  
> Adrian Hunter | 4651| 0.4%  
  
Krzysztof Kozlowski is the developer who contributed the most patches to 5.18; this work consisted mainly of device-tree updates. Matthew Wilcox managed to get another set of [folio patches](<https://lwn.net/Articles/893512/>) merged. Christoph Hellwig continues to massively refactor the block and filesystem layers. Geert Uytterhoeven contributed a large set of Renesas pin-control improvements, and Ville Syrjälä did a lot of work on the Intel i915 graphics driver. 

In the "changed lines" column, Leo Li added over 200,000 lines with just five patches adding register definitions for the AMD graphics driver — and Qingqing Zhuo added nearly 200,000 more. Ian Rogers made a number of improvements to the `perf` tool, Alan Kao contributed a single patch removing the nds32 architecture, and Ming Qian contributed a set of Amphion media drivers. 

The top testers and reviewers of patches were: 

> Test and review credits in 5.18  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 155| 11.7%  
> Damien Le Moal | 77| 5.8%  
> Konrad Jankowski | 54| 4.1%  
> David Howells | 53| 4.0%  
> Mike Marshall | 53| 4.0%  
> Gurucharan | 38| 2.9%  
> Marc Zyngier | 32| 2.4%  
> Vladimir Murzin | 32| 2.4%  
> Randy Dunlap | 21| 1.6%  
> Jiri Olsa | 17| 1.3%  
> Julian Grahsl | 16| 1.2%  
> Yihang Li | 15| 1.1%  
> | Reviewed-by  
> ---  
> Rob Herring | 217| 2.7%  
> Christoph Hellwig | 204| 2.6%  
> Andy Shevchenko | 143| 1.1%  
> AngeloGioacchino Del Regno | 110| 1.4%  
> Stephen Boyd | 103| 1.3%  
> Pierre-Louis Bossart | 103| 1.3%  
> Alex Deucher | 98| 1.2%  
> Krzysztof Kozlowski | 96| 1.2%  
> Hans de Goede | 91| 1.1%  
> Péter Ujfalusi | 88| 1.1%  
> Jani Nikula | 86| 1.1%  
> Himanshu Madhani | 85| 1.1%  
  
Daniel Wheeler continues to receive the most test credits, having applied Tested-by tags to many AMD graphics-driver patches. It's worth noting that Wheeler posts occasional [summaries](<https://lore.kernel.org/all/DM6PR12MB3529DA896040C5B7AA54B22D9CCF9@DM6PR12MB3529.namprd12.prod.outlook.com/>) describing the testing that has been done. Damien Le Moal tested many of the folio patches. and Konrad Jankowski regularly tests Intel network-driver patches. 

Turning to the review column, Rob Herring routinely reviews device-tree patches. Christoph Hellwig reviewed patches in the block and filesystem subsystems — and a number of the folio patches as well. Andy Shevchenko reviewed many driver patches, mostly in the I2C, GPIO, and pin-control subsystems. 

In the past it has been easy to be cynical about these numbers; they didn't capture much of the test and review activity happening in the community and were easily gamed. There is still surely a lot of work going on that is not reflected above, but it would be hard to argue that the testers and reviewers on these lists don't belong there. Perhaps this reflects a greater understanding of the value of these activities on the part of developers and (especially) their employers. 

Whether the same can be said for bug reporting will be left for the reader to decide: 

> Top bug-report credits for 5.18  
> ---  
> kernel test robot | 232| 19.3%  
> Zeal Robot | 76| 6.3%  
> Syzbot| 72| 6.0%  
> Abaci | 62| 5.2%  
> Dan Carpenter | 29| 2.4%  
> Hulk Robot | 27| 2.2%  
> Stephen Rothwell | 26| 2.2%  
> Igor Zhbanov | 19| 1.6%  
> Randy Dunlap | 12| 1.0%  
> Rob Herring | 9| 0.7%  
  
Bug reporting is clearly a job for robots these days. But note that, while 2,249 5.18 patches were backported to the 5.17 stable updates (so far), only 1,075 contained Reported-by tags. That would suggest that that just over half of the fixes being applied do not carry those tags and that, probably, a number of bug reports are going without credit. 

The employers contributing most actively to this development cycle were: 

> Most active 5.18 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1708| 11.4%  
> (Unknown)| 1155| 7.7%  
> Red Hat| 958| 6.4%  
> Google| 886| 5.9%  
> (None)| 818| 5.5%  
> AMD| 781| 5.2%  
> Linaro| 560| 3.7%  
> Huawei Technologies| 471| 3.1%  
> Facebook| 446| 3.0%  
> NVIDIA| 396| 2.6%  
> (Consultant)| 363| 2.4%  
> SUSE| 344| 2.3%  
> IBM| 334| 2.2%  
> Oracle| 325| 2.2%  
> Arm| 294| 2.0%  
> Renesas Electronics| 262| 1.8%  
> MediaTek| 249| 1.7%  
> NXP Semiconductors| 236| 1.6%  
> Canonical| 227| 1.5%  
> Microchip Technology| 201| 1.3%  
> | By lines changed  
> ---  
> AMD| 467642| 39.9%  
> Intel| 107081| 9.1%  
> Google| 103801| 8.8%  
> (Unknown)| 49669| 4.2%  
> Linaro| 29631| 2.5%  
> Red Hat| 28807| 2.5%  
> (None)| 27989| 2.4%  
> NXP Semiconductors| 21418| 1.8%  
> NVIDIA| 19203| 1.6%  
> MediaTek| 18980| 1.6%  
> Facebook| 16036| 1.4%  
> Andes Technology| 15814| 1.3%  
> (Consultant)| 14314| 1.2%  
> Huawei Technologies| 13483| 1.1%  
> IBM| 11960| 1.0%  
> Microchip Technology| 11853| 1.0%  
> Renesas Electronics| 11427| 1.0%  
> SUSE| 10128| 0.9%  
> Canonical| 8984| 0.8%  
> Fungible| 8791| 0.7%  
  
As usual, there are few surprises here. 

[![\[Patch-flow plot\]](https://static.lwn.net/images/2022/treeplot-5.18.svg)](<https://lwn.net/Articles/895911/>)

#### Patch flow and signed tags

The illegible plot on the right (click to be able to actually read it) shows the paths taken by patches into the mainline kernel. Each box represents a Git repository, with the vectors showing the movement of patches from one repository to the next. This plot, which was generated by the `treeplot` utility from the gitdm collection of hacks (available from `git://git.lwn.net/gitdm.git`), provides an overall picture of how code moves through the maintainer community. 

That picture remains relatively flat; most maintainers push their changes directly to Linus Torvalds. There is, however, a steady growth in the role of intermediate repositories, with the biggest ones handling the networking, graphics, system-on-chip, and character driver subsystems. The plot is a schematic diagram of the machine that has allowed the kernel process to scale to its current size — and, presumably, beyond. 

The color of each vector indicates whether that repository is using signed tags on patches being pushed to the next level in the hierarchy; red lines indicate the lack of such a tag. The use of GPG signatures on tags allows a receiving maintainer to verify that a pull request was created by the person it claims to be from. If all pull requests include signed tags, it will be significantly harder for an attacker to convince a maintainer to pull from a malicious branch. 

As has been [documented here](<https://lwn.net/Articles/737093/>) over the years, that universal use of signed tags has been slow to happen. Recently, though, Torvalds has become more insistent, with explicit [requests](<https://lwn.net/ml/linux-kernel/CAHk-=wj4eF-HZc+uuVcD4EjLW_QN7_8OZ5gtAC9_6qY1-ZK4rg@mail.gmail.com/>) to recalcitrant maintainers to get with the program. The end result is that, for 5.18, only 714 patches did not come from a signed tag — and 565 of those were directly applied by Torvalds and didn't arrive via a Git repository at all. So, at the top level of the tree, the switch to using signed tags is nearly complete — a mere 11 years after the practice was adopted. Some of the mid-level maintainers are still clearly not requiring signed tags on pull requests, though, so there are still some holes in the process. 

#### Older bugs

Many of the patches applied to 5.18 fix bugs; how old are those bugs? One way of approximating an answer to that question is to look at how many fixes showing up in the stable updates were first applied to 5.18. A bug fix, one would expect, will not be backported beyond the release that introduced the bug in the first place. The results for 5.18 are: 

> Release| Backports  
> ---|---  
> 5.17 (Mar 2022) | 2,249  
> 5.15 (Oct 2021) | 1,762  
> 5.10 (Dec 2020) | 1,185  
> 5.4 (Nov 2019) | 756  
> 4.19 (Oct 2018) | 532  
> 4.14 (Nov 2017) | 422  
> 4.9 (Dec 2016) | 331  
  
As can be seen above, 331 fixes (so far) have been ported from 5.18 all the way back to the 4.9 kernel, which was released over five years ago. In other words, after more than five years of intensive fixing (stable updates to 4.9 have added nearly 22,000 fixes), we are still fixing nearly five bugs in 4.9 every day. We'll get that kernel right one of these years, probably just before its end of life date. 

To summarize, the kernel machine continues to move at high speed. Lots of bugs are being fixed and, beyond doubt, lots more are being introduced. The end result continues to be the kernel that we all rely on.
