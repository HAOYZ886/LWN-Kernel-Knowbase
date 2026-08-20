---
title: Development statistics for 6.8
url: https://lwn.net/Articles/964106/
date: "March 11, 2024"
category: "Releases-6.8"
author: "By Jonathan Corbet March 11, 2024"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 11, 2024

The 6.8 kernel was [released](<https://lwn.net/Articles/964977/>) on March 10 after a typical, nine-week development cycle. Over this time, 1,938 developers contributed 14,405 non-merge changesets, making 6.8 into a slower cycle than 6.7 (but busier than 6.6), with the lowest number of developers participating since the 6.5 release. Still, there was a lot going on during this cycle; read on for some of the details. 

Of the developers contributing to 6.8, 245 appeared for the first time. The most active developers in this cycle were: 

> Most active 6.8 developers  
> ---  
> | By changesets  
> ---  
> Uwe Kleine-König | 368| 2.6%  
> Kent Overstreet | 317| 2.2%  
> Lucas De Marchi | 189| 1.3%  
> Krzysztof Kozlowski | 182| 1.3%  
> Dmitry Baryshkov | 148| 1.0%  
> Matt Roper | 135| 0.9%  
> Andy Shevchenko | 133| 0.9%  
> Andrii Nakryiko | 129| 0.9%  
> Matthew Brost | 115| 0.8%  
> Matthew Wilcox | 113| 0.8%  
> David Howells | 108| 0.7%  
> Arnd Bergmann | 104| 0.7%  
> Matthew Auld | 102| 0.7%  
> Randy Dunlap | 102| 0.7%  
> Jakub Kicinski | 94| 0.7%  
> Neil Armstrong | 90| 0.6%  
> Alexander Viro | 90| 0.6%  
> Thomas Zimmermann | 83| 0.6%  
> Christoph Hellwig | 80| 0.6%  
> Konrad Dybcio | 79| 0.5%  
> | By changed lines  
> ---  
> Arnd Bergmann | 59205| 7.3%  
> Matthew Brost | 46142| 5.7%  
> Jakub Kicinski | 37553| 4.6%  
> Sarah Walker | 29771| 3.7%  
> Neil Armstrong | 21336| 2.6%  
> Rajendra Nayak | 16235| 2.0%  
> Thomas Zimmermann | 14881| 1.8%  
> Andrii Nakryiko | 12938| 1.6%  
> Kent Overstreet | 12617| 1.6%  
> Darrick J. Wong | 12403| 1.5%  
> David Howells | 10224| 1.3%  
> Nas Chung | 10207| 1.3%  
> Ping-Ke Shih | 8007| 1.0%  
> Shinas Rasheed | 8006| 1.0%  
> Dmitry Safonov | 7938| 1.0%  
> Lucas De Marchi | 7324| 0.9%  
> Vlastimil Babka | 5377| 0.7%  
> Peter Griffin | 5263| 0.7%  
> Donald Robson | 4911| 0.6%  
> Dmitry Baryshkov | 4873| 0.6%  
  
In the changesets column, Uwe Kleine-König once again ends up on top, mostly for ongoing work refactoring platform drivers. Kent Overstreet is not far behind, though, as he works to stabilize bcachefs (and also did a bit of include-file rationalization). Lucas De Marchi worked on the new Intel Xe graphics driver, Krzysztof Kozlowski worked mostly with devicetree files, and Dmitry Baryshkov worked extensively with Qualcomm drivers. 

Arnd Bergmann, as usual, worked all over the kernel tree; he landed at the top of the "changed lines" column by removing a number of old and unloved WiFi drivers. Matthew Brost did a lot of work with the Xe driver. Jakub Kicinski removed a bunch of machine-generated, netlink-related code, Sarah Walker added the PowerVR/IMG GPU driver, and Neil Armstrong added a number of Qualcomm clock-controller drivers. 

The top testers and reviewers this time around were: 

> Test and review credits in 6.8  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 151| 14.0%  
> Pucha Himasekhar Reddy | 51| 4.7%  
> Hyeonggon Yoo | 31| 2.9%  
> Fuad Tabba | 23| 2.1%  
> Arnaldo Carvalho de Melo | 23| 2.1%  
> Philipp Hortmann | 23| 2.1%  
> David Rientjes | 21| 1.9%  
> Andrew Halaney | 19| 1.8%  
> Hans de Goede | 16| 1.5%  
> Jeremi Piotrowski | 16| 1.5%  
> Randy Dunlap | 15| 1.4%  
> Neil Armstrong | 14| 1.3%  
> | Reviewed-by  
> ---  
> Krzysztof Kozlowski | 226| 2.4%  
> Matt Roper | 214| 2.3%  
> Simon Horman | 210| 2.2%  
> Konrad Dybcio | 204| 2.1%  
> Christoph Hellwig | 195| 2.1%  
> Matthew Brost | 189| 2.0%  
> Rodrigo Vivi | 176| 1.9%  
> Lucas De Marchi | 167| 1.8%  
> Dmitry Baryshkov | 121| 1.3%  
> AngeloGioacchino Del Regno | 115| 1.2%  
> Hans de Goede | 107| 1.1%  
> Jan Kara | 98| 1.0%  
  
As usual, Daniel Wheeler tests many of the driver patches coming out of AMD, while Pucha Himasekhar Reddy performs a similar function within Intel. Hyeonggon Yoo, instead, has made a habit of testing memory-management patches coming from a number of developers. On the review side, Krzysztof Kozlowski reviewed large numbers of devicetree patches; Matt Roper's review was focused mostly on Xe patches. Konrad Dybcio also reviewed devicetree patches, Simon Horman worked in the networking subsystem, and Christoph Hellwig looked at a lot of block-layer patches. 

Looking at Signed-off-by tags applied by developers other than the author of a patch reveals who handled the patch after it was posted; it shows who the first-level maintainers are. In 6.8, the pattern of non-author signoffs was a bit different than usual: 

> Non-author signoffs in 6.8  
> ---  
> | Individuals  
> ---  
> Rodrigo Vivi | 955| 7.1%  
> Greg Kroah-Hartman | 797| 6.0%  
> Andrew Morton| 601| 4.5%  
> Jakub Kicinski | 565| 4.2%  
> David S. Miller | 537| 4.0%  
> Alex Deucher | 500| 3.7%  
> Mark Brown | 485| 3.6%  
> Bjorn Andersson | 443| 3.3%  
> Alexei Starovoitov | 279| 2.1%  
> Hans Verkuil | 260| 1.9%  
> Kalle Valo | 214| 1.6%  
> Arnaldo Carvalho de Melo | 177| 1.3%  
> Martin K. Petersen | 164| 1.2%  
> Paolo Abeni | 153| 1.1%  
> Takashi Iwai | 149| 1.1%  
> Herbert Xu | 147| 1.1%  
> Shawn Guo | 131| 1.0%  
> Palmer Dabbelt | 130| 1.0%  
> Mauro Carvalho Chehab | 128| 1.0%  
> Vinod Koul | 127| 0.9%  
> | Employers  
> ---  
> Intel| 2260| 16.9%  
> Red Hat| 1434| 10.7%  
> Linaro| 1337| 10.0%  
> Google| 1326| 9.9%  
> Meta| 1048| 7.8%  
> Linux Foundation| 860| 6.4%  
> Qualcomm| 740| 5.5%  
> AMD| 653| 4.9%  
> SUSE| 409| 3.1%  
> (Unknown)| 377| 2.8%  
> (None)| 278| 2.1%  
> Cisco| 260| 1.9%  
> NVIDIA| 249| 1.9%  
> Oracle| 203| 1.5%  
> Microsoft| 200| 1.5%  
> Huawei Technologies| 171| 1.3%  
> IBM| 150| 1.1%  
> Collabora| 147| 1.1%  
> Rivos| 132| 1.0%  
> (Consultant)| 124| 0.9%  
  
Rodrigo Vivi is not a name that comes quickly to mind when thinking about kernel maintainers (even for those of us who think about such things). In what is getting to be a common theme, the reason for his presence at the top of this list is that he is the maintainer who manages patches for the new Xe driver. Other than that, the busiest maintainers are the usual crowd that one would expect to see on that list. The Xe work also put Intel at the top of the signoffs list — though the Xe patches account for less than half of the total handled by Intel maintainers. 

As has been the case for years, over half of the patches going into the kernel pass through the hands of developers working for just five companies. 

Speaking of companies, 219 companies were identified as supporting work on the 6.8 kernel; the most active of those were: 

> Most active 6.8 employers  
> ---  
> | By changesets  
> ---  
> Intel| 2527| 17.5%  
> (Unknown)| 1087| 7.5%  
> Linaro| 1084| 7.5%  
> Google| 878| 6.1%  
> Red Hat| 871| 6.0%  
> (None)| 757| 5.3%  
> AMD| 657| 4.6%  
> Pengutronix| 416| 2.9%  
> SUSE| 372| 2.6%  
> Meta| 368| 2.6%  
> Oracle| 346| 2.4%  
> NVIDIA| 266| 1.8%  
> Qualcomm| 261| 1.8%  
> Huawei Technologies| 237| 1.6%  
> IBM| 224| 1.6%  
> Collabora| 167| 1.2%  
> Broadcom| 142| 1.0%  
> Arm| 141| 1.0%  
> Bootlin| 135| 0.9%  
> Renesas Electronics| 132| 0.9%  
> | By lines changed  
> ---  
> Intel| 151009| 18.7%  
> Linaro| 115647| 14.3%  
> Meta| 59065| 7.3%  
> (Unknown)| 52084| 6.4%  
> Red Hat| 43378| 5.4%  
> Imagination Technologies| 34692| 4.3%  
> Qualcomm| 30115| 3.7%  
> SUSE| 27574| 3.4%  
> (None)| 22259| 2.8%  
> Google| 22067| 2.7%  
> AMD| 21853| 2.7%  
> Oracle| 19462| 2.4%  
> Realtek| 12587| 1.6%  
> Marvell| 10869| 1.3%  
> Bootlin| 8978| 1.1%  
> MediaTek| 8449| 1.0%  
> NVIDIA| 8163| 1.0%  
> Arista Networks| 7955| 1.0%  
> Ideas on Board| 7429| 0.9%  
> ST Microelectronics| 7057| 0.9%  
  
Intel dominates the by-changesets list — and would be at the top even without the Xe contribution. The 6.7 kernel showed a spike in contributions from unaffiliated developers; that number has reverted to something close to its long-term mean in 6.8, though. Otherwise, these numbers are about the same as they usually are. 

#### Fixes

Commits fixing a bug should contain a `Fixes` tag identifying the commit that introduced the bug; that practice helps in the understanding of the problem and informs the backporting effort for the stable releases. In 6.8, 2,582 commits contained a total of 2732 `Fixes` tags identifying 2,292 commits in 90 releases. Of those tags, 533 identified other 6.8 commits, and thus do not refer to bugs that made it into a released kernel. 

The distribution of the remaining tags is shown in the following table. The "Fixed" column indicates the number of commits in the named release that were fixed by commits in 6.8, while "By" gives the number of commits in 6.8 fixing that release. 

> Releases fixed in v6.8  
> ---  
> Release| Commits  
> Fixed| By  
> [v6.7](<https://lwn.net/Articles/964401/#v6.7>) | 241 | 282 | 282  
> [v6.6](<https://lwn.net/Articles/964401/#v6.6>) | 137 | 164 | 164  
> [v6.5](<https://lwn.net/Articles/964401/#v6.5>) | 125 | 146 | 146  
> [v6.4](<https://lwn.net/Articles/964401/#v6.4>) | 84 | 103 | 103  
> [v6.3](<https://lwn.net/Articles/964401/#v6.3>) | 83 | 91 | 91  
> [v6.2](<https://lwn.net/Articles/964401/#v6.2>) | 74 | 82 | 82  
> [v6.1](<https://lwn.net/Articles/964401/#v6.1>) | 50 | 51 | 51  
> [v6.0](<https://lwn.net/Articles/964401/#v6.0>) | 70 | 73 | 73  
> [v5.19](<https://lwn.net/Articles/964401/#v5.19>) | 53 | 54 | 54  
> [v5.18](<https://lwn.net/Articles/964401/#v5.18>) | 45 | 45 | 45  
> [v5.17](<https://lwn.net/Articles/964401/#v5.17>) | 27 | 28 | 28  
> [v5.16](<https://lwn.net/Articles/964401/#v5.16>) | 43 | 43 | 43  
> [v5.15](<https://lwn.net/Articles/964401/#v5.15>) | 32 | 35 | 35  
> [v5.14](<https://lwn.net/Articles/964401/#v5.14>) | 22 | 26 | 26  
> [v5.13](<https://lwn.net/Articles/964401/#v5.13>) | 33 | 35 | 35  
> [v5.12](<https://lwn.net/Articles/964401/#v5.12>) | 27 | 31 | 31  
> [v5.11](<https://lwn.net/Articles/964401/#v5.11>) | 30 | 35 | 35  
> [v5.10](<https://lwn.net/Articles/964401/#v5.10>) | 23 | 26 | 26  
> [v5.9](<https://lwn.net/Articles/964401/#v5.9>) | 25 | 27 | 27  
> [v5.8](<https://lwn.net/Articles/964401/#v5.8>) | 26 | 29 | 29  
> [v5.7](<https://lwn.net/Articles/964401/#v5.7>) | 26 | 30 | 30  
> [v5.6](<https://lwn.net/Articles/964401/#v5.6>) | 31 | 34 | 34  
> [v5.5](<https://lwn.net/Articles/964401/#v5.5>) | 16 | 16 | 16  
> [v5.4](<https://lwn.net/Articles/964401/#v5.4>) | 15 | 25 | 25  
> [v5.3](<https://lwn.net/Articles/964401/#v5.3>) | 25 | 25 | 25  
> [v5.2](<https://lwn.net/Articles/964401/#v5.2>) | 11 | 11 | 11  
> [v5.1](<https://lwn.net/Articles/964401/#v5.1>) | 12 | 19 | 19  
> [v5.0](<https://lwn.net/Articles/964401/#v5.0>) | 11 | 12 | 12  
> [v4.20](<https://lwn.net/Articles/964401/#v4.20>) | 25 | 30 | 30  
> [v4.19](<https://lwn.net/Articles/964401/#v4.19>) | 18 | 21 | 21  
> [v4.18](<https://lwn.net/Articles/964401/#v4.18>) | 13 | 13 | 13  
> [v4.17](<https://lwn.net/Articles/964401/#v4.17>) | 11 | 10 | 10  
> [v4.16](<https://lwn.net/Articles/964401/#v4.16>) | 14 | 16 | 16  
> [v4.15](<https://lwn.net/Articles/964401/#v4.15>) | 11 | 11 | 11  
> [v4.14](<https://lwn.net/Articles/964401/#v4.14>) | 9 | 10 | 10  
> [v4.13](<https://lwn.net/Articles/964401/#v4.13>) | 5 | 5 | 5  
> [v4.12](<https://lwn.net/Articles/964401/#v4.12>) | 7 | 8 | 8  
> [v4.11](<https://lwn.net/Articles/964401/#v4.11>) | 11 | 11 | 11  
> [v4.10](<https://lwn.net/Articles/964401/#v4.10>) | 14 | 14 | 14  
> [v4.9](<https://lwn.net/Articles/964401/#v4.9>) | 7 | 7 | 7  
> [v4.8](<https://lwn.net/Articles/964401/#v4.8>) | 14 | 14 | 14  
> [v4.7](<https://lwn.net/Articles/964401/#v4.7>) | 8 | 9 | 9  
> [v4.6](<https://lwn.net/Articles/964401/#v4.6>) | 7 | 7 | 7  
> [v4.5](<https://lwn.net/Articles/964401/#v4.5>) | 4 | 4 | 4  
> [v4.4](<https://lwn.net/Articles/964401/#v4.4>) | 6 | 6 | 6  
> [v4.3](<https://lwn.net/Articles/964401/#v4.3>) | 9 | 9 | 9  
> [v4.2](<https://lwn.net/Articles/964401/#v4.2>) | 9 | 10 | 10  
> [v4.1](<https://lwn.net/Articles/964401/#v4.1>) | 6 | 7 | 7  
> [v4.0](<https://lwn.net/Articles/964401/#v4.0>) | 1 | 1 | 1  
> [v3.19](<https://lwn.net/Articles/964401/#v3.19>) | 4 | 9 | 9  
> [v3.18](<https://lwn.net/Articles/964401/#v3.18>) | 9 | 9 | 9  
> [v3.17](<https://lwn.net/Articles/964401/#v3.17>) | 7 | 7 | 7  
> [v3.16](<https://lwn.net/Articles/964401/#v3.16>) | 13 | 13 | 13  
> [v3.15](<https://lwn.net/Articles/964401/#v3.15>) | 6 | 6 | 6  
> [v3.14](<https://lwn.net/Articles/964401/#v3.14>) | 4 | 5 | 5  
> [v3.13](<https://lwn.net/Articles/964401/#v3.13>) | 3 | 6 | 6  
> [v3.12](<https://lwn.net/Articles/964401/#v3.12>) | 6 | 6 | 6  
> [v3.11](<https://lwn.net/Articles/964401/#v3.11>) | 6 | 6 | 6  
> [v3.10](<https://lwn.net/Articles/964401/#v3.10>) | 11 | 16 | 16  
> [v3.9](<https://lwn.net/Articles/964401/#v3.9>) | 4 | 4 | 4  
> [v3.8](<https://lwn.net/Articles/964401/#v3.8>) | 4 | 4 | 4  
> [v3.7](<https://lwn.net/Articles/964401/#v3.7>) | 4 | 4 | 4  
> [v3.6](<https://lwn.net/Articles/964401/#v3.6>) | 2 | 2 | 2  
> [v3.5](<https://lwn.net/Articles/964401/#v3.5>) | 4 | 4 | 4  
> [v3.4](<https://lwn.net/Articles/964401/#v3.4>) | 3 | 3 | 3  
> [v3.3](<https://lwn.net/Articles/964401/#v3.3>) | 5 | 6 | 6  
> [v3.2](<https://lwn.net/Articles/964401/#v3.2>) | 5 | 5 | 5  
> [v3.1](<https://lwn.net/Articles/964401/#v3.1>) | 3 | 3 | 3  
> [v3.0](<https://lwn.net/Articles/964401/#v3.0>) | 4 | 4 | 4  
> [v2.6.39](<https://lwn.net/Articles/964401/#v2.6.39>) | 6 | 6 | 6  
> [v2.6.38](<https://lwn.net/Articles/964401/#v2.6.38>) | 4 | 6 | 6  
> [v2.6.37](<https://lwn.net/Articles/964401/#v2.6.37>) | 3 | 3 | 3  
> [v2.6.36](<https://lwn.net/Articles/964401/#v2.6.36>) | 2 | 2 | 2  
> [v2.6.35](<https://lwn.net/Articles/964401/#v2.6.35>) | 2 | 2 | 2  
> [v2.6.34](<https://lwn.net/Articles/964401/#v2.6.34>) | 4 | 4 | 4  
> [v2.6.33](<https://lwn.net/Articles/964401/#v2.6.33>) | 2 | 2 | 2  
> [v2.6.31](<https://lwn.net/Articles/964401/#v2.6.31>) | 1 | 1 | 1  
> [v2.6.30](<https://lwn.net/Articles/964401/#v2.6.30>) | 3 | 3 | 3  
> [v2.6.29](<https://lwn.net/Articles/964401/#v2.6.29>) | 2 | 3 | 3  
> [v2.6.28](<https://lwn.net/Articles/964401/#v2.6.28>) | 2 | 2 | 2  
> [v2.6.27](<https://lwn.net/Articles/964401/#v2.6.27>) | 2 | 2 | 2  
> [v2.6.26](<https://lwn.net/Articles/964401/#v2.6.26>) | 2 | 2 | 2  
> [v2.6.25](<https://lwn.net/Articles/964401/#v2.6.25>) | 2 | 2 | 2  
> [v2.6.23](<https://lwn.net/Articles/964401/#v2.6.23>) | 1 | 1 | 1  
> [v2.6.22](<https://lwn.net/Articles/964401/#v2.6.22>) | 3 | 5 | 5  
> [v2.6.18](<https://lwn.net/Articles/964401/#v2.6.18>) | 1 | 1 | 1  
> [v2.6.17](<https://lwn.net/Articles/964401/#v2.6.17>) | 2 | 2 | 2  
> [v2.6.13](<https://lwn.net/Articles/964401/#v2.6.13>) | 1 | 1 | 1  
> [v2.6.12](<https://lwn.net/Articles/964401/#v2.6.12>) | 1 | 23 | 23  
  
Thus, 6.8 contained 23 commits with fixes tags identifying a single commit in 2.6.12 that needed a lot of fixing; that is, of course, the initial commit made at the beginning of the Git era. It has been almost 19 years, but we're still fixing bugs that went in prior to the adoption of Git. 

The pattern shown above is typical for a kernel release; while a lot of the bugs fixed were introduced within the last year, there are also vast numbers of bugs that have lurked in the kernel for far longer. 

#### In conclusion

As a final note: the Xe driver, first merged for 6.8, figures strongly in the statistics for this cycle; it is worth looking just a bit more at this work to see what is involved in adding a new graphics driver to the kernel. The Xe driver accounted for 1,041 changesets in this development cycle. Those commits were contributed by 70 developers, 66 of whom work for Intel (with a few still sticking to their Habana Labs email addresses). Their work added about 60,000 lines of code to the kernel. 

Once upon a time, such a code contribution would have been huge news; in 2024, it draws little attention outside of the community that is interested in graphics drivers. Such is the nature of contemporary kernel development, where the addition of major new code components is a routine event. As of this writing, there are over 11,600 changesets waiting in linux-next for the 6.9 merge window to open, so it seems that the flow will not be stopping soon; keep your eyes on LWN to see what those commits bring.
