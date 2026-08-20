---
title: A look at some 5.17 development statistics
url: https://lwn.net/Articles/887559/
date: "March 21, 2022"
category: "Releases-5.17"
author: "By Jonathan Corbet March 21, 2022"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 21, 2022

At the conclusion of the 5.17 development cycle, 13,038 non-merge changesets had found their way into the mainline repository. That is a lower level of activity than was seen for 5.16 (14,190 changesets) but well above 5.15 (12,337). In other words, this was a fairly typical kernel release. That is true in terms of where the work that made up the release came from as well. 

The changes in 5.17 were contributed by 1,900 developers, down from the 1,988 seen in 5.16. Of those developers, 268 made their first kernel contributions in this cycle. The most active developers this time around were: 

> Most active 5.17 developers  
> ---  
> | By changesets  
> ---  
> Christoph Hellwig | 168| 1.3%  
> Eric Dumazet | 150| 1.2%  
> Mauro Carvalho Chehab | 142| 1.1%  
> Hans de Goede | 139| 1.1%  
> Andy Shevchenko | 132| 1.0%  
> Martin Kaiser | 132| 1.0%  
> Christophe Jaillet | 125| 1.0%  
> Ville Syrjälä | 123| 0.9%  
> Thierry Reding | 114| 0.9%  
> Sean Christopherson | 109| 0.8%  
> Thomas Gleixner | 105| 0.8%  
> Matthew Wilcox| 101| 0.8%  
> Andrii Nakryiko | 97| 0.7%  
> Nicholas Piggin | 96| 0.7%  
> Michael Straube | 92| 0.7%  
> David Howells | 89| 0.7%  
> Lad Prabhakar | 86| 0.7%  
> Dmitry Osipenko | 82| 0.6%  
> Rob Herring | 76| 0.6%  
> Vladimir Oltean | 74| 0.6%  
> | By changed lines  
> ---  
> David Howells | 26567| 4.4%  
> Thierry Reding | 16552| 2.7%  
> Christoph Hellwig | 10734| 1.8%  
> Luiz Augusto von Dentz | 10106| 1.7%  
> Mauro Carvalho Chehab | 10010| 1.7%  
> Vinod Koul | 9363| 1.5%  
> Zong-Zhe Yang | 8135| 1.3%  
> Svyatoslav Ryhel | 7204| 1.2%  
> Horatiu Vultur | 6962| 1.1%  
> Hans de Goede | 6537| 1.1%  
> Chengchang Tang | 6255| 1.0%  
> AngeloGioacchino Del Regno| 6198| 1.0%  
> Andrzej Pietrasiewicz | 6035| 1.0%  
> Dmitry Osipenko | 6013| 1.0%  
> Amit Cohen | 5949| 1.0%  
> Daniel Bristot de Oliveira | 5598| 0.9%  
> Jie Wang | 5553| 0.9%  
> Ville Syrjälä | 5451| 0.9%  
> Jacob Keller | 4943| 0.8%  
> Emmanuel Grumbach | 4615| 0.8%  
  
Christoph Hellwig continues to do extensive refactoring, mostly in the block and filesystem layers; once again, this work has made him the top changeset contributor. Eric Dumazet, as always, has been busy making the networking stack work better; he also added the [reference-count tracking infrastructure](<https://lwn.net/Articles/877603/>) during this cycle. Mauro Carvalho Chehab does most of his work in the media subsystem, Hans de Goede works mostly in the graphics layer (including adding generic support for privacy screens this time around), and Andy Shevchenko contributed numerous cleanups throughout the driver subsystem. 

David Howells topped out the "lines changed" column by rewriting and replacing the caching layer used by network filesystems. Thierry Reding contributed a lot of Tegra SoC hardware support, and Luiz Augusto von Dentz worked extensively on the Bluetooth host-controller interface code. 

The most active testers and reviewers of patches this time around were: 

> Test and review credits in 5.17  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 105| 10.1%  
> Gurucharan G | 51| 4.9%  
> Nishanth Menon | 41| 3.9%  
> Michael Kelley | 31| 3.0%  
> Konrad Jankowski | 31| 3.0%  
> Sebastian Andrzej Siewior | 21| 2.0%  
> Juergen Gross | 19| 1.8%  
> Wolfram Sang | 17| 1.6%  
> Bean Huo | 16| 1.5%  
> Tony Brelinski | 14| 1.3%  
> Valentin Schneider | 11| 1.1%  
> Arnaldo Carvalho de Melo | 10| 1.0%  
> Sachin Sant | 10| 1.0%  
> | Reviewed-by  
> ---  
> Rob Herring | 175| 2.9%  
> Christoph Hellwig | 138| 2.3%  
> Andy Shevchenko | 104| 1.7%  
> David Sterba | 96| 1.6%  
> Jason Gunthorpe | 91| 1.5%  
> Pierre-Louis Bossart | 84| 1.4%  
> Jeff Layton | 83| 1.4%  
> Kai Vehmanen | 68| 1.1%  
> Greg Kroah-Hartman | 66| 1.1%  
> Krzysztof Kozlowski | 65| 1.1%  
> Ranjani Sridharan | 65| 1.1%  
> Darrick J. Wong | 62| 1.0%  
> Ville Syrjälä | 59| 1.0%  
  
Many of these names have appeared in these tables for a while now; perhaps the biggest change is the appearance of Andy Shevchenko, whose Reviewed-by tag appears on patches throughout the driver subsystem. 

Work on 5.17 was supported by 245 employers that we were able to identify; that is, again, a typical number for recent kernels. The most active employers were: 

> Most active 5.17 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1510| 11.6%  
> (Unknown)| 975| 7.5%  
> Red Hat| 894| 6.9%  
> Google| 878| 6.7%  
> (None)| 572| 4.4%  
> AMD| 490| 3.8%  
> Huawei Technologies| 481| 3.7%  
> Linaro| 452| 3.5%  
> NVIDIA| 438| 3.4%  
> SUSE| 425| 3.3%  
> Facebook| 388| 3.0%  
> (Consultant)| 381| 2.9%  
> IBM| 316| 2.4%  
> Oracle| 265| 2.0%  
> Renesas Electronics| 245| 1.9%  
> Arm| 217| 1.7%  
> Alibaba| 205| 1.6%  
> NXP Semiconductors| 186| 1.4%  
> Qualcomm| 185| 1.4%  
> Collabora| 150| 1.2%  
> | By lines changed  
> ---  
> Intel| 76222| 12.6%  
> Red Hat| 56565| 9.3%  
> (Unknown)| 39769| 6.6%  
> NVIDIA| 32721| 5.4%  
> Huawei Technologies| 30039| 5.0%  
> Google| 24971| 4.1%  
> (None)| 21917| 3.6%  
> AMD| 21133| 3.5%  
> Linaro| 20866| 3.4%  
> Qualcomm| 20116| 3.3%  
> (Consultant)| 18397| 3.0%  
> SUSE| 16841| 2.8%  
> Facebook| 14405| 2.4%  
> Collabora| 13845| 2.3%  
> Realtek| 11255| 1.9%  
> Microchip Technology| 9613| 1.6%  
> IBM| 8974| 1.5%  
> NXP Semiconductors| 8039| 1.3%  
> SoMainline| 7789| 1.3%  
> Renesas Electronics| 7767| 1.3%  
  
Once again, there are few surprises here. 

#### Old bugs?

While a development series like 5.17 brings a long list of new features, it also includes fixes for older bugs. There are various ways of calculating just how old those bugs are, but one metric has the advantage of being relatively easy to calculate: how many patches in 5.17 have been backported to the stable updates for previous kernels? The 4.19 kernel was released in October 2018, for example, so any patches backported to the 4.19 stable updates can be seen as being fixes for problems that are at least that old. 

It's a fairly straightforward task to look at the mainline commit ID for each commit in a stable series and see if it is a 5.17 commit or not. Indeed, this can be done for older kernels as well; the results look like this: 

> Stable update | Patches from  
> ---|---  
> 5.17 | 5.16 | 5.15 | 5.14  
> Count| Pct | Count| Pct | Count| Pct | Count| Pct  
> 5.16.14| 2,323| 99.3% | |  | |  | |   
> 5.15.28| 1,993| 45.0% | 2,361| 53.8% | |  | |   
> 5.10.105| 1,241| 10.2% | 1,342| 10.9% | 1,374| 11.1% | 1,864| 15.1%  
> 5.4.184| 739| 4.2% | 869| 4.9% | 860| 4.9% | 1,120| 6.4%  
> 4.19.234| 528| 2.4% | 599| 2.7% | 564| 2.6% | 760| 3.5%  
> 4.14.271| 426| 1.9% | 477| 2.1% | 439| 1.9% | 565| 2.1%  
> 4.9.306| 334| 1.6% | 368| 1.7% | 339| 1.6% | 434| 2.0%  
> 4.4.302| 133| 0.7% | 289| 1.5% | 269| 1.4% | 329| 1.7%  
  
For each mainline/stable-update pair, the entries in the table show how many patches were backported to the stable series from that mainline release, and the percentage of all the patches in that stable series that came from that mainline release. Thus we see, for example, that essentially all of the patches backported to 5.16 came from 5.17 — an unsurprising observation. (It is not 100% because there are always a few patches that are not directly backported, or are just version tags). 

Reading down the columns shows that, as time goes on, the number of bugs fixed in the older stable updates does decrease, as one would expect. But it definitely does not drop to zero; patches were still being backported from 5.17 to the 4.4 kernel (released over six years ago) right up until that kernel stopped receiving support. Reading across the columns suggests that there is nothing special about 5.17; every mainline release is fixing a steady stream of bugs that have been present for years. 

Of course, there are any number of important caveats here. For example, backported patches could be fixing bugs in other backported patches, in which case the bugs are more recent than it would seem. As [documented here](<https://lwn.net/Articles/812231/>) in the past, the regression rate in stable kernels runs anywhere from about 3% to 12%, depending on how one counts. There is also the fact that not all backported patches are bug fixes; some add device IDs or improve performance, for example. 

There are also fixes for hardware bugs. For example, about 30 of the just-backported 5.17 patches address the [branch history injection](<https://lwn.net/Articles/887326/>) Spectre vulnerability. It is not fair to chalk those up as fixes for bugs in the older kernels, but they are problems that needed to be fixed regardless. 

These factors all suggest that the above numbers could be adjusted but are not fundamentally wrong. 

Overall, 5.17 was another typical, relatively boring kernel development cycle. The kernel-creation machine continues to crank out releases on a predictable schedule. As of this writing, linux-next contains over 12,800 changesets waiting to be dumped into the mainline for 5.18, so it does not look like the process will stop anytime soon. 

#### A postscript

Given the stability of the kernel's development process, these reports have become increasingly uninteresting over time; there is not a lot of news to be found here. Given that, your editor is, once again, questioning the value of producing them for every kernel release. Increasingly, they seem like a bunch of boilerplate with some side investigations tossed in to try to make them more interesting. Might it be more useful to discontinue this practice in favor of, say, a full-year report on the occasion of each long-term-stable release? Please feel free to let us know, via the comments or email to `lwn@lwn.net`, if you have an opinion on this matter.
