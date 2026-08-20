---
title: Development statistics for 6.12
url: https://lwn.net/Articles/997959/
date: "November 18, 2024"
category: "Releases-6.12"
author: "By Jonathan Corbet November 18, 2024"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 18, 2024

Linus Torvalds [released the 6.12 kernel](<https://lwn.net/ml/all/CAHk-=wgtGkHshfvaAe_O2ntnFBH3EprNk1juieLmjcF2HBwBgQ@mail.gmail.com/>) on November 17, as expected. This development cycle, the last for 2024, brought 13,344 non-merge changesets into the mainline kernel; that made it a relatively slow cycle from this perspective, but 6.12 includes a long list of significant new features. The time has come to look at where those changes came from, and to look at the year-long LTS cycle as well. 

The 6.12 kernel included work from 2,074 developers; this is not a record (that is 2,090 in 6.2), but is close. Of those developers, 335 made their first contribution to the kernel during this cycle; that _is_ a record for the Git era (and probably before as well). The most active developers during this cycle were: 

> Most active 6.12 developers  
> ---  
> | By changesets  
> ---  
> Krzysztof Kozlowski | 225| 1.7%  
> Kent Overstreet | 186| 1.4%  
> Tejun Heo | 131| 1.0%  
> Jinjie Ruan | 123| 0.9%  
> Javier Carrasco | 109| 0.8%  
> Sean Christopherson | 108| 0.8%  
> Andy Shevchenko | 107| 0.8%  
> Takashi Iwai | 106| 0.8%  
> Alex Deucher | 95| 0.7%  
> Nuno Sa | 94| 0.7%  
> Christoph Hellwig | 90| 0.7%  
> Frank Li | 89| 0.7%  
> Jani Nikula | 88| 0.7%  
> Rob Herring | 85| 0.6%  
> Matthew Wilcox | 85| 0.6%  
> Ian Rogers | 83| 0.6%  
> Namhyung Kim | 75| 0.6%  
> Christian Brauner | 74| 0.6%  
> Christophe JAILLET | 73| 0.5%  
> Hongbo Li | 72| 0.5%  
> | By changed lines  
> ---  
> Cezary Rojewski | 22850| 3.7%  
> Yevgeny Kliteynik | 17704| 2.8%  
> Samson Tam | 14305| 2.3%  
> Tejun Heo | 14224| 2.3%  
> Herbert Xu | 11867| 1.9%  
> Nikita Shubin | 9270| 1.5%  
> Pavitrakumar M | 8378| 1.3%  
> Philipp Hortmann | 7690| 1.2%  
> Eddie James | 7138| 1.1%  
> Lorenzo Stoakes | 6919| 1.1%  
> Dmitry Torokhov | 6667| 1.1%  
> Alexandre Mergnat | 6385| 1.0%  
> Kent Overstreet | 6309| 1.0%  
> David Howells | 5435| 0.9%  
> Harald Freudenberger | 5124| 0.8%  
> Takashi Iwai | 4922| 0.8%  
> Deven Bowers | 4873| 0.8%  
> Inochi Amaoto | 4739| 0.8%  
> Junfeng Guo | 4503| 0.7%  
> Chuck Lever | 4416| 0.7%  
  
Krzysztof Kozlowski continued a long-running effort to refactor low-level device code and devicetree bindings. Kent Overstreet is also working on a long-running project — the effort to stabilize the bcachefs filesystem. Tejun Heo contributed the [extended scheduler class](<https://lwn.net/Articles/974387/>). Jinjie Ruan and Javier Carrasco both contributed a lot of cleanups in the driver subsystem. 

In the "lines changed" column, Cezary Rojewski removed a number of old audio drivers. Yevgeny Kliteynik added a bunch of functionality to the mlx5 network-interface driver. Samson Tam added some new features to the AMD graphics driver, and Herbert Xu reverted a set of cryptographic-driver patches that were not properly submitted. 

There were Reviewed-by tags in 48% of the commits merged for 6.12, while just under 10% of the commits in this release included Tested-by tags. The top testers and reviewers this time around were: 

> Test and review credits in 6.12  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 198| 14.6%  
> Philipp Hortmann | 58| 4.3%  
> Arnaldo Carvalho de Melo | 55| 4.0%  
> Rafal Romanowski | 33| 2.4%  
> Alexander Sverdlin | 30| 2.2%  
> Jonathan Cameron | 25| 1.8%  
> Valentin Schneider | 23| 1.7%  
> Ojaswin Mujoo | 22| 1.6%  
> Alibek Omarov | 20| 1.5%  
> Zi Yan | 19| 1.4%  
> Pucha Himasekhar Reddy | 18| 1.3%  
> Andreas Kemnade | 18| 1.3%  
> Alice Ryhl | 17| 1.3%  
> Björn Töpel | 17| 1.3%  
> | Reviewed-by  
> ---  
> Simon Horman | 210| 2.5%  
> Krzysztof Kozlowski | 180| 2.2%  
> Andrew Lunn | 131| 1.6%  
> David Sterba | 116| 1.4%  
> Jan Kara | 109| 1.3%  
> Darrick J. Wong | 99| 1.2%  
> Christoph Hellwig | 98| 1.2%  
> Jeff Layton | 97| 1.2%  
> Josef Bacik | 95| 1.1%  
> Geert Uytterhoeven | 93| 1.1%  
> Jonathan Cameron | 90| 1.1%  
> Rob Herring | 87| 1.0%  
> Andy Shevchenko | 82| 1.0%  
> Konrad Dybcio | 81| 1.0%  
  
The testing side is dominated, as usual, by people who seem to do that work as their primary job; one exception would be Arnaldo Carvalho de Melo, who tests a lot of `perf` patches as the maintainer before applying them. Simon Horman was the most prolific reviewer this time around, adding his tag to over three network-subsystem patches every day of this development cycle. 

Work on 6.12 was supported by 218 employers that we were able to identify — a typical number. The most active employers were: 

> Most active 6.12 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1240| 9.3%  
> (Unknown)| 1173| 8.8%  
> Google| 957| 7.2%  
> AMD| 810| 6.1%  
> Huawei Technologies| 791| 5.9%  
> (None)| 672| 5.0%  
> Red Hat| 651| 4.9%  
> Linaro| 618| 4.6%  
> Meta| 480| 3.6%  
> NVIDIA| 382| 2.9%  
> SUSE| 361| 2.7%  
> Oracle| 262| 2.0%  
> Renesas Electronics| 254| 1.9%  
> IBM| 249| 1.9%  
> Arm| 241| 1.8%  
> NXP Semiconductors| 236| 1.8%  
> (Consultant)| 229| 1.7%  
> Qualcomm| 175| 1.3%  
> Microsoft| 159| 1.2%  
> Linutronix| 140| 1.0%  
> | By lines changed  
> ---  
> Intel| 68687| 11.0%  
> (Unknown)| 52196| 8.3%  
> AMD| 44794| 7.2%  
> Google| 42921| 6.9%  
> Red Hat| 38609| 6.2%  
> Meta| 30757| 4.9%  
> NVIDIA| 30555| 4.9%  
> IBM| 20294| 3.2%  
> Oracle| 18201| 2.9%  
> Linaro| 17513| 2.8%  
> (None)| 17146| 2.7%  
> SUSE| 15243| 2.4%  
> BayLibre| 14470| 2.3%  
> Qualcomm| 11740| 1.9%  
> NXP Semiconductors| 11214| 1.8%  
> Microsoft| 10858| 1.7%  
> Huawei Technologies| 10181| 1.6%  
> Realtek| 9941| 1.6%  
> YADRO| 9274| 1.5%  
> Arm| 8545| 1.4%  
  
This list seldom contains surprises, and 6.12 follows in the usual pattern. One notable point is the appearance of Linutronix; that is a result of the merging of the final realtime patches and a fair amount of related refactoring work. 

#### The longer cycle

While the kernel development cycle takes nine or ten weeks, almost without exception, it is a rare user who installs all of those releases. Instead, an increasing portion of the user body is running one of the long-term-support (LTS) releases and the stable updates that are built on top of them. By convention, the final release of the year becomes an LTS release. 

As a result, there is an argument to be made that the real kernel development cycle takes about one year — the time that elapses between the LTS releases that are actually deployed by users. The 6.12 release, being the last release of 2024, is thus the end of that longer cycle, so there may be value in looking at the statistics for the full year. 

Since the release of the last LTS kernel (6.6), the development community has created six releases, incorporating 86,715 non-merge changesets from 5,111 developers. The most active developers over the whole year were: 

> Most active 6.7-12 developers  
> ---  
> | By changesets  
> ---  
> Kent Overstreet | 3972| 4.6%  
> Uwe Kleine-König | 1596| 1.8%  
> Krzysztof Kozlowski | 1339| 1.5%  
> Andy Shevchenko | 817| 0.9%  
> Jani Nikula | 676| 0.8%  
> Dmitry Baryshkov | 637| 0.7%  
> Christoph Hellwig | 634| 0.7%  
> Ville Syrjälä | 581| 0.7%  
> Johannes Berg | 568| 0.7%  
> Matthew Wilcox | 537| 0.6%  
> Eric Dumazet | 489| 0.6%  
> Ian Rogers | 474| 0.5%  
> Geert Uytterhoeven | 471| 0.5%  
> Darrick J. Wong | 446| 0.5%  
> Thomas Zimmermann | 431| 0.5%  
> Kees Cook | 401| 0.5%  
> Arnd Bergmann | 395| 0.5%  
> Sean Christopherson | 381| 0.4%  
> Jeff Johnson | 378| 0.4%  
> Jakub Kicinski | 374| 0.4%  
> | By changed lines  
> ---  
> Kent Overstreet | 259293| 5.1%  
> Aurabindo Pillai | 228673| 4.5%  
> Hawking Zhang | 152950| 3.0%  
> Ian Rogers | 133772| 2.6%  
> Qingqing Zhuo | 101474| 2.0%  
> Dmitry Baryshkov | 88968| 1.7%  
> Hamza Mahfooz | 73053| 1.4%  
> Arnd Bergmann | 71392| 1.4%  
> Ard Biesheuvel | 70780| 1.4%  
> Ben Li | 68066| 1.3%  
> Lang Yu | 66939| 1.3%  
> Philipp Hortmann | 63036| 1.2%  
> Matthew Sakai | 58728| 1.2%  
> Darrick J. Wong | 55467| 1.1%  
> Matthew Brost | 51447| 1.0%  
> Jakub Kicinski | 47447| 0.9%  
> Matthew Wilcox | 40377| 0.8%  
> Neil Armstrong | 36116| 0.7%  
> Sarah Walker | 29771| 0.6%  
> David Howells | 27675| 0.5%  
  
Unsurprisingly, these results are consistent with what has been seen over the course of the last year. Overstreet, it should be noted, found his way to the top of both lists through the merger of a body of work that was developed out-of-tree for years. The main source of new lines of code coming into the kernel, though, was the seemingly endless stream of machine-generated header files for the amdgpu driver. 

The top testers and reviewers over the longer cycle were: 

> Test and review credits in 6.7-12  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 1136| 14.1%  
> Philipp Hortmann | 244| 3.0%  
> Pucha Himasekhar Reddy | 214| 2.7%  
> Arnaldo Carvalho de Melo | 124| 1.5%  
> Michael Kelley | 101| 1.3%  
> Neil Armstrong | 99| 1.2%  
> Sohil Mehta | 92| 1.1%  
> Rafal Romanowski | 85| 1.1%  
> Nicolin Chen | 81| 1.0%  
> Randy Dunlap | 64| 0.8%  
> Björn Töpel | 57| 0.7%  
> Babu Moger | 56| 0.7%  
> Geert Uytterhoeven | 54| 0.7%  
> Sujai Buvaneswaran | 54| 0.7%  
> Guenter Roeck | 51| 0.6%  
> Kees Cook | 50| 0.6%  
> Helge Deller | 50| 0.6%  
> Johan Hovold | 49| 0.6%  
> Nathan Chancellor | 47| 0.6%  
> Shameer Kolothum | 44| 0.5%  
> | Reviewed-by  
> ---  
> Simon Horman | 1146| 2.1%  
> Christoph Hellwig | 1009| 1.9%  
> Krzysztof Kozlowski | 1002| 1.9%  
> Konrad Dybcio | 826| 1.5%  
> Dmitry Baryshkov | 697| 1.3%  
> AngeloGioacchino Del Regno | 657| 1.2%  
> David Sterba | 626| 1.2%  
> Andy Shevchenko | 611| 1.1%  
> Rodrigo Vivi | 574| 1.1%  
> Ilpo Järvinen | 550| 1.0%  
> Andrew Lunn | 536| 1.0%  
> Rob Herring| 534| 1.0%  
> Geert Uytterhoeven | 525| 1.0%  
> Kees Cook | 465| 0.9%  
> Matt Roper | 451| 0.8%  
> Linus Walleij | 437| 0.8%  
> Jani Nikula | 430| 0.8%  
> Darrick J. Wong | 426| 0.8%  
> Jeff Layton | 424| 0.8%  
> Hawking Zhang | 418| 0.8%  
  
The most active employers (out of the 361 total) over the longer cycle were: 

> Most active 6.7-12 employers  
> ---  
> | By changesets  
> ---  
> Intel| 11356| 13.1%  
> (None)| 6881| 7.9%  
> Google| 5920| 6.8%  
> (Unknown)| 5668| 6.5%  
> AMD| 5233| 6.0%  
> Linaro| 5112| 5.9%  
> Red Hat| 4863| 5.6%  
> Huawei Technologies| 2459| 2.8%  
> SUSE| 2319| 2.7%  
> Meta| 2207| 2.5%  
> Oracle| 1986| 2.3%  
> Pengutronix| 1871| 2.2%  
> Qualcomm| 1792| 2.1%  
> NVIDIA| 1713| 2.0%  
> IBM| 1612| 1.9%  
> Renesas Electronics| 1574| 1.8%  
> (Consultant)| 1227| 1.4%  
> Arm| 1178| 1.4%  
> NXP Semiconductors| 916| 1.1%  
> Texas Instruments| 781| 0.9%  
> | By lines changed  
> ---  
> AMD| 918483| 18.1%  
> Intel| 540531| 10.6%  
> Google| 378278| 7.4%  
> (None)| 352401| 6.9%  
> Linaro| 314793| 6.2%  
> Red Hat| 308732| 6.1%  
> (Unknown)| 292949| 5.8%  
> Meta| 150897| 3.0%  
> Oracle| 136086| 2.7%  
> Qualcomm| 108629| 2.1%  
> NVIDIA| 94799| 1.9%  
> SUSE| 86590| 1.7%  
> Realtek| 78260| 1.5%  
> Emerson| 63036| 1.2%  
> IBM| 61320| 1.2%  
> Collabora| 58147| 1.1%  
> Renesas Electronics| 56839| 1.1%  
> Huawei Technologies| 50113| 1.0%  
> NXP Semiconductors| 41451| 0.8%  
> Microsoft| 38985| 0.8%  
  
Intel has cemented its position as the most prolific contributor of changesets over this year, with nearly double the number of the next company (Google) on the list. Otherwise, though, this list looks similar to [the 6.6 version](<https://lwn.net/Articles/948970/>) at the end of the last long cycle. 

All told, the kernel's development process continues to incorporate changes and bring in new developers at a high rate (though that rate has been stable for the last few years). As of this writing, there are over 10,000 changes in linux-next waiting for the 6.13 merge window to open, so there is plenty of work to start the next development cycle (and the next year-long LTS cycle). As always, LWN will be there to tell you how it goes.
