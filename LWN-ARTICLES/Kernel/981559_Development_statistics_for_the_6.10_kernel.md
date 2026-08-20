---
title: Development statistics for the 6.10 kernel
url: https://lwn.net/Articles/981559/
date: "July 15, 2024"
category: "Releases-6.10"
author: "By Jonathan Corbet July 15, 2024"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 15, 2024

The 6.10 kernel was [released](<https://lwn.net/ml/all/CAHk-=wjV_O2g_K19McjGKrxFxMFDqex+fyGcKc3uac1ft_O2gg@mail.gmail.com/>) on July 14 after a nine-week development cycle. This time around, 13,312 non-merge changesets were pulled into the mainline repository — the lowest changeset count since 5.17 in early 2022. Longstanding tradition says that it is time for LWN to gather some statistics on where the new code for 6.10 came from and how it got to the mainline; read on for the details. 

As a reminder, much of the information that follows (and much that doesn't) can be had at any time from the [LWN kernel source database](</ksdb/>), which is available to LWN subscribers. 

A total of 1,918 developers contributed to 6.10; that is lower than recent releases but essentially equal to the 1,921 contributors to 6.5 in August of last year. Of those developers, 242 made their first kernel contribution in 6.10. The most active contributors to this release were: 

> Most active 6.10 developers  
> ---  
> | By changesets  
> ---  
> Krzysztof Kozlowski | 357| 2.7%  
> Kent Overstreet | 270| 2.0%  
> Andy Shevchenko | 226| 1.7%  
> Uwe Kleine-König | 176| 1.3%  
> Darrick J. Wong | 160| 1.2%  
> Jani Nikula | 133| 1.0%  
> Matthew Wilcox| 125| 0.9%  
> Ville Syrjälä | 122| 0.9%  
> Eric Dumazet | 120| 0.9%  
> Ian Rogers | 102| 0.8%  
> Hans de Goede | 101| 0.8%  
> Dmitry Baryshkov | 101| 0.8%  
> Christoph Hellwig | 94| 0.7%  
> Takashi Iwai | 94| 0.7%  
> Geert Uytterhoeven | 90| 0.7%  
> Arnd Bergmann | 87| 0.7%  
> Damien Le Moal | 87| 0.7%  
> Wolfram Sang | 82| 0.6%  
> Namhyung Kim | 78| 0.6%  
> Pierre-Louis Bossart | 77| 0.6%  
> | By changed lines  
> ---  
> Dmitry Baryshkov | 64636| 10.0%  
> Darrick J. Wong | 23781| 3.7%  
> Philipp Hortmann | 18687| 2.9%  
> Bingbu Cao | 14333| 2.2%  
> Boris Brezillon | 14090| 2.2%  
> Wedson Almeida Filho | 10335| 1.6%  
> Kent Overstreet | 9144| 1.4%  
> David Howells | 8347| 1.3%  
> Bitterblue Smith | 6117| 0.9%  
> Hans de Goede | 5821| 0.9%  
> Namhyung Kim | 5792| 0.9%  
> Ian Rogers | 5592| 0.9%  
> Arnd Bergmann | 5492| 0.8%  
> Benjamin Tissoires | 4924| 0.8%  
> Michal Wajdeczko | 4804| 0.7%  
> Tushar Vyavahare | 4647| 0.7%  
> Shahab Vahedi | 4629| 0.7%  
> Fiona Klute | 4485| 0.7%  
> Akhil R | 4316| 0.7%  
> Jordan Rife | 4135| 0.6%  
  
Krzysztof Kozlowski was the leading contributor of commits to the 6.10 kernel, producing over five changes for every day of this release cycle; this work was focused on refactoring throughout the driver subsystems and devicetree changes. Kent Overstreet's work was mostly focused on fixes for the bcachefs filesystem, but he also implemented much of the [memory-allocation profiling](<https://lwn.net/Articles/974380/>) subsystem. Andy Shevchenko contributed cleanups throughout the driver tree, Uwe Kleine-König continued the driver-refactoring work that has led to 1,883 changes since the 6.4 release, and Darrick Wong did a lot of work on the XFS filesystem, mostly focused on the in-progress online-repair functionality. 

In the lines-changed column, Dmitry Baryshkov removed a number of header files that can be automatically generated at build time. Philipp Hortmann removed a couple of unloved drivers from the staging tree. Bingbu Cao added a number of Intel media drivers, and Boris Brezillon contributed [the Panthor driver](<https://www.collabora.com/news-and-blog/news-and-events/release-the-panthor.html>) for Arm Mali GPUs. 

The top testers and reviewers this time around were: 

> Test and review credits in 6.10  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 141| 11.7%  
> Kees Cook | 46| 3.8%  
> Pucha Himasekhar Reddy | 43| 3.6%  
> Hans Holmberg | 28| 2.3%  
> Dennis Maisenbacher | 28| 2.3%  
> Jarkko Sakkinen | 25| 2.1%  
> Mark Pearson | 24| 2.0%  
> Arnaldo Carvalho de Melo | 22| 1.8%  
> Philipp Hortmann | 22| 1.8%  
> Atish Patra | 17| 1.4%  
> Björn Töpel | 16| 1.3%  
> Sujai Buvaneswaran | 15| 1.2%  
> Pierre-Louis Bossart | 14| 1.2%  
> Ryan Roberts | 14| 1.2%  
> | Reviewed-by  
> ---  
> Christoph Hellwig | 303| 3.4%  
> Simon Horman | 237| 2.7%  
> Bard Liao | 165| 1.9%  
> Andy Shevchenko | 149| 1.7%  
> Krzysztof Kozlowski | 123| 1.4%  
> Konrad Dybcio | 118| 1.3%  
> David Sterba | 116| 1.3%  
> AngeloGioacchino Del Regno | 105| 1.2%  
> Dmitry Baryshkov | 104| 1.2%  
> Jani Nikula | 93| 1.1%  
> Kees Cook | 90| 1.0%  
> Rob Herring | 86| 1.0%  
> Hans de Goede | 85| 1.0%  
> Andrew Lunn | 85| 1.0%  
  
These lists do not change much from one release to the next; the developers who are able to find the time for this sort of work tend to be in it for the long haul. 

A total of 203 companies (that we know of) supported work on the 6.10 kernel; the most active of those were: 

> Most active 6.10 employers  
> ---  
> | By changesets  
> ---  
> Intel| 2031| 15.3%  
> Google| 980| 7.4%  
> Red Hat| 922| 6.9%  
> (Unknown)| 891| 6.7%  
> Linaro| 838| 6.3%  
> (None)| 690| 5.2%  
> AMD| 610| 4.6%  
> Oracle| 443| 3.3%  
> Meta| 424| 3.2%  
> SUSE| 327| 2.5%  
> IBM| 321| 2.4%  
> Huawei Technologies| 301| 2.3%  
> Qualcomm| 247| 1.9%  
> Renesas Electronics| 246| 1.8%  
> Pengutronix| 221| 1.7%  
> (Consultant)| 214| 1.6%  
> NVIDIA| 205| 1.5%  
> Arm| 160| 1.2%  
> NXP Semiconductors| 135| 1.0%  
> Collabora| 131| 1.0%  
> | By lines changed  
> ---  
> Intel| 88245| 13.6%  
> Linaro| 86382| 13.3%  
> (Unknown)| 71831| 11.1%  
> Red Hat| 46754| 7.2%  
> Google| 37411| 5.8%  
> Oracle| 28846| 4.4%  
> AMD| 25793| 4.0%  
> Collabora| 21917| 3.4%  
> (None)| 19151| 2.9%  
> Meta| 16586| 2.6%  
> Microsoft| 14554| 2.2%  
> NVIDIA| 10990| 1.7%  
> IBM| 10350| 1.6%  
> ST Microelectronics| 8179| 1.3%  
> Bootlin| 7746| 1.2%  
> Qualcomm| 7636| 1.2%  
> Realtek| 7509| 1.2%  
> SUSE| 7268| 1.1%  
> Arm| 7019| 1.1%  
> NXP Semiconductors| 6910| 1.1%  
  
For the most part, this list looks as it usually does. It is worth noting, though, that Intel contributed more than twice as many changesets to 6.10 as any other company — over 15% of the total. 

#### Tree flatness

[![\[Tree plot\]](https://static.lwn.net/images/2024/treeplot-6.10.svg)](<https://lwn.net/Articles/981742/>) [One of the suggested topics](<https://lwn.net/ml/all/nycvar.YFH.7.76.2406041151590.24940@cbobk.fhfr.pm>) for the upcoming Maintainers Summit (to be held on September 17) was whether the merge tree is too flat. In other words, are there too many maintainers sending pull requests directly to Linus Torvalds rather than going through intermediate-level maintainers? Using the `treeplot` utility from gitdm, one can see what this tree looks like. An unreadable version appears to the right; clicking on it (preferably on a system with a large monitor) will shed more light. 

One conclusion that can be drawn is that, while the tree does indeed seem flat, with a lot of trees feeding directly into the mainline (note that 63 such trees have been collapsed into one line in the diagram), there is also a fair amount of work going through more than one repository as well. There would appear to be a slow trend toward adding levels to the hierarchy — but it is indeed slow. 

Another way to look at this is to create a histogram showing how many levels of repository each commit went through: 

> Trees transited by each commit  
> ---  
> Trees| Commits  
> 0 | 26 | 26  
> 1 | 8,242 | 8,242  
> 2 | 4,817 | 4,817  
> 3 | 227 | 227  
  
The 26 commits that passed through zero trees before landing in the mainline were those committed directly by Torvalds. Nearly two-thirds of all commits went through a single tree before going upstream, but a substantial number also went through at least one other tree first. Whether that indicates an overly flat tree is hard to say. 

One other thing worth noting: the trees shown in red in the plot are those that did not include a signed tag authenticating their creators. Torvalds has been increasingly insistent that pull requests should point to signed tags over the last few years, with the result that nearly every pull request he does now includes a signature. Most subsystem maintainers clearly do the same, but there still an occasional unsigned pull that finds its way upstream. 

In any case, it seems evident from the above that, even if the process could always benefit from improvements, it is working reasonably well. The kernel community continues to absorb changes at a high rate and put out releases on a predictable schedule. As of this writing, there are just over 12,000 changesets poised to flow into the mainline from linux-next, so the 6.11 kernel (which will likely be released on September 15, putting the next merge window in the middle of several important conferences) will be busy as well.
