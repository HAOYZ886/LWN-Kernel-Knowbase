---
title: Development statistics for 6.13
url: https://lwn.net/Articles/1004998/
date: "January 20, 2025"
category: "Releases-6.13"
author: "By Jonathan Corbet January 20, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 20, 2025

The 6.13 development cycle ended on January 19 with the [release](<https://lwn.net/ml/all/CAHk-=wiprabAQcCwb3qNhrT5P50MJNqunC9JU5v99kdvM-2rsg@mail.gmail.com/>) of the 6.13 kernel. This cycle was, on its surface, one of the slowest we have seen in some time; the LWN merge-window summaries ([part 1](<https://lwn.net/Articles/998623/>), [part 2](<https://lwn.net/Articles/998990/>)) and [the KernelNewbies 6.13 page](<https://kernelnewbies.org/Linux_6.13>) can be consulted for a refresher on all it contains. Here, instead, we will take our usual look at where all of those changes came from. 

The 6.13 kernel cycle brought in 12,928 non-merge changesets from 2,001 developers. The changeset count is noteworthy for being the lowest since 5.15 (12,377 changesets) in 2021. If one looks at a plot of changeset traffic for each kernel release (taken from [the LWN Kernel Source Database](</ksdb/>)), one sees: 

> ![\[Changeset counts per
  kernel release\]](https://static.lwn.net/images/2025/6.13-csets.svg)

The immediate impression is that kernel development activity has gone into a decline since the 6.7 release (January 2024) hit a record with 17,284 changesets. Interestingly, though, a plot of the number of developers participating in each release tells a slightly different story: 

> ![\[Developers per
  kernel release\]](https://static.lwn.net/images/2025/6.13-devs.svg)

The 2,001 developers who contributed to 6.13 are somewhat short of the record (2,090) set with 6.2 in 2023, but the previous release (6.12, at 2,074 developers) came close. Kernel developers may be merging fewer changesets at the moment, but that does not necessarily mean that less work is being done and, in any case, there are as many people working on the kernel as there has ever been. 

The most active developers this time around were: 

> Most active 6.13 developers  
> ---  
> | By changesets  
> ---  
> Thomas Zimmermann | 185| 1.4%  
> Andy Shevchenko | 158| 1.2%  
> Sean Christopherson | 147| 1.1%  
> Jani Nikula | 142| 1.1%  
> Darrick J. Wong | 113| 0.9%  
> Christoph Hellwig | 112| 0.9%  
> Dmitry Baryshkov | 111| 0.9%  
> Javier Carrasco | 111| 0.9%  
> Thomas Weißschuh | 102| 0.8%  
> Ian Rogers | 99| 0.8%  
> Dr. David Alan Gilbert | 91| 0.7%  
> Thomas Gleixner | 90| 0.7%  
> Philipp Hortmann | 84| 0.6%  
> Ville Syrjälä | 82| 0.6%  
> Masahiro Yamada | 81| 0.6%  
> Uwe Kleine-König | 80| 0.6%  
> Dmitry Torokhov | 80| 0.6%  
> Bartosz Golaszewski | 79| 0.6%  
> Mark Brown | 79| 0.6%  
> Kuniyuki Iwashima | 78| 0.6%  
> | By changed lines  
> ---  
> Philipp Hortmann | 76407| 10.0%  
> Jan Kara | 32830| 4.3%  
> Dave Penkler | 29327| 3.8%  
> Johannes Berg | 26104| 3.4%  
> Dmitry Baryshkov | 11847| 1.6%  
> Bitterblue Smith | 10591| 1.4%  
> Daniel Machon | 9878| 1.3%  
> Hans Verkuil | 9490| 1.2%  
> Marek Vasut | 8282| 1.1%  
> Detlev Casanova | 8121| 1.1%  
> Andy Shevchenko | 7755| 1.0%  
> Darrick J. Wong | 7566| 1.0%  
> Konrad Dybcio | 6661| 0.9%  
> Taniya Das | 5891| 0.8%  
> Ian Rogers | 5831| 0.8%  
> Dr. David Alan Gilbert | 5570| 0.7%  
> Ivaylo Ivanov | 5469| 0.7%  
> Tomi Valkeinen | 5278| 0.7%  
> Jani Nikula | 5038| 0.7%  
> Zong-Zhe Yang | 4832| 0.6%  
  
Thomas Zimmermann topped the by-changesets column with a long list of improvements throughout the graphics subsystem. Andy Shevchenko contributed refactoring and cleanups over much of the driver subsystem. Sean Christopherson continued the ongoing refactoring of the KVM subsystem. Jani Nikula contributed many changes to the Intel i915 graphics driver, and Darrick Wong worked extensively in the XFS filesystem, with much of that work going toward online filesystem-checking functionality. 

In the lines-changed column, Philipp Hortmann removed a number of old wireless-network drivers. Jan Kara removed the ReiserFS filesystem. Dave Penkler added the GPIB driver subsystem to the staging tree. Johannes Berg also joined the wireless-driver-removal party, and Dmitry Baryshkov added support for a number of Qualcomm clocks. 

If the 6.13 development was a bit quieter than usual, some of the reasons why can be seen in the above list. The extensive refactoring work that has created large changeset counts in previous kernels is mostly absent this time around. The massive amdgpu header-file contributions that have bloated past kernels are also not present in 6.13. Instead, we are seeing the steady pace of development that is always happening, but which can be obscured in the statistics by those larger changes. 

Just over half (51.3%) of the commits merged for 6.13 included at least one Reviewed-by tag, but only 8% had Tested-by tags. The top testers and reviewers in 6.13 were: 

> Test and review credits in 6.13  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 148| 11.9%  
> Dmitry Osipenko | 85| 6.8%  
> Alex Bennée | 55| 4.4%  
> Neil Armstrong | 47| 3.8%  
> Pucha Himasekhar Reddy | 30| 2.4%  
> Gary Guo | 19| 1.5%  
> Nathan Chancellor | 15| 1.2%  
> Randy Dunlap | 15| 1.2%  
> Nicolin Chen | 14| 1.1%  
> Serge Semin | 13| 1.0%  
> Steev Klimaszewski | 13| 1.0%  
> Philipp Zabel | 12| 1.0%  
> Chris Healy | 12| 1.0%  
> Leo Yan | 11| 0.9%  
> Geert Uytterhoeven | 10| 0.8%  
> Rob Clark | 10| 0.8%  
> Rajneesh Bhardwaj | 10| 0.8%  
> Arnaldo Carvalho de Melo | 10| 0.8%  
> | Reviewed-by  
> ---  
> Christoph Hellwig | 216| 2.5%  
> Simon Horman | 211| 2.4%  
> Krzysztof Kozlowski | 197| 2.3%  
> Dmitry Baryshkov | 119| 1.4%  
> Ilpo Järvinen | 112| 1.3%  
> Eric Dumazet | 111| 1.3%  
> Rodrigo Vivi | 95| 1.1%  
> Rob Herring | 94| 1.1%  
> Darrick J. Wong | 91| 1.1%  
> Andy Shevchenko | 91| 1.1%  
> AngeloGioacchino Del Regno | 86| 1.0%  
> Geert Uytterhoeven | 84| 1.0%  
> Neil Armstrong | 81| 0.9%  
> Ville Syrjälä | 81| 0.9%  
> David Sterba | 78| 0.9%  
> Jason Gunthorpe | 78| 0.9%  
> Christian König | 76| 0.9%  
> Laurent Pinchart | 75| 0.9%  
  
These results do not change much from one release to the next; the people who do this work are in it for the long haul. LWN subscribers may see [this KSDB page](<https://lwn.net/ksdb/releases/v6.13/by_tags>) for more information on the testers, reviewers, and bug reporters for 6.13. 

Work on 6.13 was supported by 212 employers (that we know of), a typical number. The top employers were: 

> Most active 6.13 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1482| 11.4%  
> (Unknown)| 1143| 8.8%  
> Google| 977| 7.5%  
> Red Hat| 698| 5.4%  
> AMD| 644| 5.0%  
> Linaro| 626| 4.8%  
> (None)| 586| 4.5%  
> SUSE| 422| 3.3%  
> Huawei Technologies| 382| 2.9%  
> Meta| 344| 2.6%  
> Oracle| 340| 2.6%  
> IBM| 316| 2.4%  
> Qualcomm| 271| 2.1%  
> Renesas Electronics| 257| 2.0%  
> NVIDIA| 231| 1.8%  
> NXP Semiconductors| 225| 1.7%  
> Linutronix| 212| 1.6%  
> Arm| 210| 1.6%  
> BayLibre| 206| 1.6%  
> (Consultant)| 199| 1.5%  
> | By lines changed  
> ---  
> (Unknown)| 91599| 12.0%  
> Intel| 82247| 10.8%  
> Emerson| 76407| 10.0%  
> SUSE| 42347| 5.5%  
> Qualcomm| 39053| 5.1%  
> Linaro| 31638| 4.1%  
> Google| 30487| 4.0%  
> AMD| 28441| 3.7%  
> Red Hat| 26727| 3.5%  
> (None)| 24370| 3.2%  
> Oracle| 15285| 2.0%  
> Meta| 15114| 2.0%  
> Collabora| 14973| 2.0%  
> IBM| 13880| 1.8%  
> Microchip Technology Inc.| 13658| 1.8%  
> Realtek| 12685| 1.7%  
> NVIDIA| 11718| 1.5%  
> BayLibre| 10654| 1.4%  
> Cisco| 9864| 1.3%  
> NXP Semiconductors| 9185| 1.2%  
  
Here, too, there are not many surprises to be found. 

There were a number of significant changes in 6.13; the addition of [lazy preemption](<https://lwn.net/Articles/994322/>) and a lot of important Rust infrastructure may, over time, come to be seen as some of the most important. But the removals this time around are also significant. ReiserFS was once the brightest light among Linux filesystems; it was the first to bring journaling, among other things. It has long since been surpassed and gone out of use, but its role should not be forgotten. 

The removal of the old wireless drivers also marks the end of an era of sorts. When Linux first started supporting wireless network interfaces, they were treated much like Ethernet interfaces with a few extra parameters to tweak. The "wireless extensions" interface was added to enable that tweaking. It did not take long to realize that wireless interfaces needed to be treated as a different class of device, and the networking subsystem moved in that direction. But the wireless extensions, as a user-facing interface, had to be supported for many years despite its inability to perform all of the necessary management functions for modern network adapters. Only with 6.13 has it been possible, finally, to remove that support. 

ReiserFS was introduced in 2.4.0.4 in 2001, and the wireless extensions came with 2.1.15 in 1996. This kernel release, too, surely includes code that will come to be seen, decades from now, as outmoded and ready for removal. But there will be code to replace it; as of this writing, there are just short of 9,000 changesets in linux-next waiting to be merged for 6.14. That release, too, seems unlikely to set any records for change volume. Even below its peak rate, though, the kernel community is a busy place.
