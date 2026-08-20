---
title: Development statistics for 6.14
url: https://lwn.net/Articles/1013892/
date: "March 25, 2025"
category: "Releases-6.14"
author: "By Jonathan Corbet March 25, 2025"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 25, 2025

By the time that Linus Torvalds [released](<https://lwn.net/Articles/1015309/>) the 6.14 kernel, 11,003 non-merge changesets had been pulled into the mainline, making this one of the smallest releases we have seen in some time. Indeed, one must go back to [the 4.0 release](<https://lwn.net/Articles/640081/>), which happened almost exactly ten years ago, to find a release with fewer changesets than 6.14. Even so, "small" is relative, and 6.14 contains a lot of significant changes. 

This kernel release included contributions from 1,897 developers; that, too, is a relatively small number — though rather larger than the 1,457 who participated in the 4.0 release. One only needs to go back to 5.15, released in 2021, to find a kernel release made with the participation of fewer developers. Of the developers contributing to 6.14, 228 were first-time contributors. 

The most active developers this time around were: 

> Most active 6.14 developers  
> ---  
> | By changesets  
> ---  
> Kent Overstreet | 275| 2.5%  
> Thomas Weißschuh | 183| 1.7%  
> Jani Nikula | 123| 1.1%  
> Sean Christopherson | 120| 1.1%  
> Dmitry Baryshkov | 106| 1.0%  
> Krzysztof Kozlowski | 105| 1.0%  
> David Howells | 97| 0.9%  
> Russell King | 94| 0.9%  
> Darrick J. Wong | 93| 0.8%  
> Jakub Kicinski | 91| 0.8%  
> Christoph Hellwig | 84| 0.8%  
> Dr. David Alan Gilbert | 80| 0.7%  
> Eric Biggers | 72| 0.7%  
> Mario Limonciello | 68| 0.6%  
> Eric Dumazet | 67| 0.6%  
> Christian Brauner | 66| 0.6%  
> SeongJae Park | 60| 0.5%  
> Steven Rostedt | 58| 0.5%  
> Claudiu Beznea | 58| 0.5%  
> Ian Rogers | 57| 0.5%  
> | By changed lines  
> ---  
> Richard Fitzgerald | 17946| 3.5%  
> Darrick J. Wong | 10807| 2.1%  
> Eric Biggers | 10622| 2.1%  
> Karan Tilak Kumar | 10174| 2.0%  
> Lizhi Hou | 9252| 1.8%  
> Kent Overstreet | 9075| 1.8%  
> David Howells | 8366| 1.6%  
> Taniya Das | 7181| 1.4%  
> Dmitry Baryshkov | 6827| 1.3%  
> Patrick Rudolph | 6089| 1.2%  
> Charlie Jenkins | 5961| 1.2%  
> Even Xu | 5169| 1.0%  
> Jani Nikula | 4965| 1.0%  
> Melody Olvera | 4841| 0.9%  
> Jie Gan | 4781| 0.9%  
> Krzysztof Kozlowski | 4769| 0.9%  
> Nuno Das Neves | 4579| 0.9%  
> Piotr Kwapulinski | 4460| 0.9%  
> SeongJae Park | 4306| 0.8%  
> Lukas Bulwahn | 4256| 0.8%  
  
The developer with the most changesets this time around is Kent Overstreet, who merged the usual set of fixes and improvements to the bcachefs filesystem; that count was increased by the fact that Overstreet missed the 6.13 development cycle. Thomas Weißschuh contributed fixes and hardening patches throughout the kernel. Jani Nikula worked extensively on the Intel i915 GPU driver, Sean Christopherson continued a long-running series of KVM improvements, and Dmitry Baryshkov contributed a set of graphics and devicetree changes. 

Looking at the "lines changed" column, Richard Fitzgerald added a set of unit tests for the Cirrus firmware drivers. Darrick Wong added a long list of improvements to the XFS filesystem. Eric Biggers did some extensive refactoring in the crypto subsystem and removed some unused drivers. Karan Tilak Kumar contributed extensively to the Cisco FNIC driver, and Lizhi Hou added a new driver for the AMD "amdxdna" AI engine. 

The percentages of patches with Reviewed-by tags dropped a little from 6.13 (to 49.2%) in this release, but the number with Tested-by tags increased to 9.3%. The top reviewers and testers this time around were: 

> Test and review credits in 6.14  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 113| 9.6%  
> Mark Pearson | 65| 5.5%  
> Arnaldo Carvalho de Melo | 40| 3.4%  
> Sergey Senozhatsky | 24| 2.0%  
> Aaron Ma | 22| 1.9%  
> Rui Zhang | 21| 1.8%  
> Manivannan Sadhasivam | 18| 1.5%  
> Fuad Tabba | 18| 1.5%  
> Choong Yong Liang | 18| 1.5%  
> James Clark | 17| 1.4%  
> Rick Wertenbroek | 16| 1.4%  
> Dirk Behme | 16| 1.4%  
> Johan Hovold | 15| 1.3%  
> Antony Antony | 15| 1.3%  
> | Reviewed-by  
> ---  
> Christoph Hellwig | 185| 2.5%  
> Ilpo Järvinen | 152| 2.1%  
> Dmitry Baryshkov | 134| 1.8%  
> Konrad Dybcio | 124| 1.7%  
> Krzysztof Kozlowski | 121| 1.6%  
> Jeff Layton | 117| 1.6%  
> Andrew Lunn | 113| 1.5%  
> Simon Horman | 112| 1.5%  
> Geert Uytterhoeven | 108| 1.5%  
> AngeloGioacchino Del Regno | 104| 1.4%  
> Mario Limonciello | 83| 1.1%  
> Mark Pearson | 71| 1.0%  
> Johannes Thumshirn | 69| 0.9%  
> Jani Nikula | 68| 0.9%  
  
Daniel Wheeler has been the top credited tester for every kernel release since 6.3; if one discounts a disappointing second-place finish in 6.2, though, he has actually held that position since the 5.12 release in 2021. Christoph Hellwig routinely shows up at the top of the Reviewed-by column. Curious LWN subscribers can learn more about the review, test, and bug-reporting credits for this release on [this KSDB page](</ksdb/releases/v6.14/by_tags>). 

Work on 6.14 was supported by 220 employers that we know of — a few more than contributed to 6.13. The most active employers supporting 6.14 were: 

> Most active 6.14 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1177| 10.7%  
> (Unknown)| 1049| 9.5%  
> Google| 959| 8.7%  
> (None)| 658| 6.0%  
> Red Hat| 620| 5.6%  
> AMD| 583| 5.3%  
> Linaro| 471| 4.3%  
> Qualcomm| 465| 4.2%  
> Meta| 346| 3.1%  
> Oracle| 327| 3.0%  
> SUSE| 231| 2.1%  
> Renesas Electronics| 221| 2.0%  
> IBM| 213| 1.9%  
> NVIDIA| 208| 1.9%  
> Huawei Technologies| 187| 1.7%  
> (Consultant)| 185| 1.7%  
> Amadeus IT GmbH| 167| 1.5%  
> NXP Semiconductors| 155| 1.4%  
> Microsoft| 153| 1.4%  
> Arm| 139| 1.3%  
> | By lines changed  
> ---  
> Intel| 46050| 8.9%  
> Google| 44221| 8.6%  
> (Unknown)| 43909| 8.5%  
> Qualcomm| 43055| 8.4%  
> AMD| 37831| 7.3%  
> Red Hat| 30166| 5.9%  
> (None)| 25576| 5.0%  
> Cirrus Logic| 18639| 3.6%  
> Linaro| 18412| 3.6%  
> Oracle| 15375| 3.0%  
> Meta| 14161| 2.7%  
> Cisco| 10296| 2.0%  
> Microsoft| 9719| 1.9%  
> NVIDIA| 9452| 1.8%  
> Bootlin| 7888| 1.5%  
> Microchip Technology Inc.| 7534| 1.5%  
> IBM| 6872| 1.3%  
> NXP Semiconductors| 6770| 1.3%  
> 9Elements| 6099| 1.2%  
> Rivos| 6007| 1.2%  
  
As usual, there are few surprises here. It is worth noting that contributions from developers working on their own time (and those with unknown employers, many of whom will also be volunteers) is up slightly over 6.13. That may be normal variation, but it may also reflect the holidays, where volunteers may have found more time to write patches while developers working on the kernel as their job took time off. 

Whether the holidays are the real reason for the relative slowness of 6.14 is not entirely clear, though. As of this writing, there are just over 12,100 non-merge changesets sitting in linux-next, most of which will be destined for the 6.15 kernel. That number exceeds the approximately 9,000 that were staged for 6.14 by a significant margin, suggesting that 6.15 will be a busier release, more in line with recent history.
