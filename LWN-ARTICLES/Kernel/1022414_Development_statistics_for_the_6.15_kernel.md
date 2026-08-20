---
title: Development statistics for the 6.15 kernel
url: https://lwn.net/Articles/1022414/
date: "May 26, 2025"
category: "Releases-6.15"
author: "By Jonathan Corbet May 26, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 26, 2025

The 6.14 kernel development cycle only brought in 11,003 non-merge changesets, making it the slowest cycle since 4.0, which was released in 2015\. The 6.15 kernel, instead, brought in 14,612 changesets, making it the busiest release since 6.7, released at the beginning of 2024. The kernel development process, in other words, is back up to full speed. The [6.15 release](<https://lwn.net/ml/all/CAHk-=wiLRW8DN8-4jmeCZH0OpO8skXOC5e6FwMfsPwGMpQYmVQ@mail.gmail.com/>) happened on May 25, so the time has come for the obligatory look at where the changes in this release came from. 

As a reminder, LWN subscribers can find this information and more, at any time, for any kernel version since 2005, in [the LWN Kernel Source Database](</ksdb/>). 

The work in 6.15 was contributed by 2,068 developers — a relatively high number, though it falls short of the record 2,090 seen in the 6.2 development cycle. There were 262 developers who made their first kernel contribution in 6.15. The most active contributors this time around were: 

> Most active 6.15 developers  
> ---  
> | By changesets  
> ---  
> Kent Overstreet | 266| 1.8%  
> Kuninori Morimoto | 191| 1.3%  
> Ville Syrjälä | 144| 1.0%  
> Andy Shevchenko | 137| 0.9%  
> Alex Deucher | 123| 0.8%  
> Nam Cao | 123| 0.8%  
> Sean Christopherson | 117| 0.8%  
> Krzysztof Kozlowski | 115| 0.8%  
> Takashi Iwai | 114| 0.8%  
> Dr. David Alan Gilbert | 111| 0.8%  
> Thomas Weißschuh | 108| 0.7%  
> Jani Nikula | 106| 0.7%  
> Pavel Begunkov | 102| 0.7%  
> Jakub Kicinski | 94| 0.6%  
> Eric Biggers | 93| 0.6%  
> Christoph Hellwig | 92| 0.6%  
> Arnd Bergmann | 91| 0.6%  
> Matthew Wilcox | 89| 0.6%  
> Ian Rogers | 89| 0.6%  
> Mario Limonciello | 87| 0.6%  
> | By changed lines  
> ---  
> Wayne Lin | 80287| 9.5%  
> Ian Rogers | 33886| 4.0%  
> Miri Korenblit | 29176| 3.4%  
> Bitterblue Smith | 26801| 3.2%  
> Andrew Donnellan | 25819| 3.0%  
> Edward Cree | 12941| 1.5%  
> Austin Zheng | 12889| 1.5%  
> Michael Ellerman | 12629| 1.5%  
> Dikshita Agarwal | 8901| 1.1%  
> Nick Chan | 8802| 1.0%  
> Nick Terrell | 8749| 1.0%  
> Kent Overstreet | 8296| 1.0%  
> Christoph Hellwig | 7202| 0.8%  
> Eric Biggers | 7012| 0.8%  
> Dr. David Alan Gilbert | 6844| 0.8%  
> Nuno Das Neves | 6419| 0.8%  
> Ivaylo Ivanov | 5938| 0.7%  
> David Howells | 5909| 0.7%  
> Alex Deucher | 5398| 0.6%  
> Matthew Brost | 5312| 0.6%  
  
Once again, the developer with the most changesets was Kent Overstreet, who continues to work on stabilizing the bcachefs filesystem. Kuninori Morimoto contributed a large set of cleanups to the sound subsystem. Ville Syrjälä worked exclusively on the Intel i915 graphics driver. Andy Shevchenko contributed small improvements throughout the driver subsystem, and Alex Deucher worked, as always, on the AMD graphics driver subsystem. 

Returning to a pattern often seen in recent years, the "lines changed" column is led by Wayne Lin, who contributed yet another set of AMD GPU header files. Ian Rogers made a number of contributions to the perf subsystem, including updating the large Intel vendor-events files. Miri Korenblit added the new "iwlmld" driver for newer Intel WiFi adapters. Bitterblue Smith added a number of RealTek WiFi driver variants, and Andrew Donnellan removed a couple of unused CXL drivers. 

The top testers and reviewers this time around were: 

> Test and review credits in 6.15  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 163| 9.2%  
> Neil Armstrong | 64| 3.6%  
> Thomas Falcon | 35| 2.0%  
> Babu Moger | 30| 1.7%  
> Shaopeng Tan | 30| 1.7%  
> Peter Newman | 30| 1.7%  
> Amit Singh Tomar | 30| 1.7%  
> Shanker Donthineni | 30| 1.7%  
> Stefan Schmidt | 28| 1.6%  
> Nicolin Chen | 25| 1.4%  
> Xiaochun Lee | 25| 1.4%  
> Venkat Rao Bagalkote | 24| 1.4%  
> Andreas Hindborg | 21| 1.2%  
> Alison Schofield | 21| 1.2%  
> Carl Worth | 21| 1.2%  
> | Reviewed-by  
> ---  
> Simon Horman | 271| 2.7%  
> Krzysztof Kozlowski | 161| 1.6%  
> Dmitry Baryshkov | 147| 1.5%  
> Geert Uytterhoeven | 112| 1.1%  
> Andrew Lunn | 109| 1.1%  
> Ilpo Järvinen | 105| 1.1%  
> Darrick J. Wong | 105| 1.1%  
> David Sterba | 102| 1.0%  
> Rob Herring (Arm) | 100| 1.0%  
> Jonathan Cameron | 97| 1.0%  
> Linus Walleij | 96| 1.0%  
> Charles Keepax | 93| 0.9%  
> Jan Kara | 88| 0.9%  
> Christoph Hellwig | 82| 0.8%  
> Jacob Keller | 81| 0.8%  
  
Daniel Wheeler retains his permanent spot as the top-credited tester; nobody else even comes close. The top reviewers are a bit different this time around, with Simon Horman reviewing just over four networking patches for every day of this development cycle. 

There were Tested-by tags in 1,411 6.15 commits (9.7% of the total), while 7,332 (50.2%) of the commits had Reviewed-by tags. 

Work on 6.15 was supported by (at least) 195 employers, a slightly smaller number than usual. The most active employers were: 

> Most active 6.15 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1755| 12.0%  
> (Unknown)| 1302| 8.9%  
> Google| 983| 6.7%  
> (None)| 930| 6.4%  
> Red Hat| 889| 6.1%  
> AMD| 881| 6.0%  
> Linaro| 645| 4.4%  
> SUSE| 549| 3.8%  
> Meta| 493| 3.4%  
> NVIDIA| 370| 2.5%  
> Huawei Technologies| 370| 2.5%  
> Renesas Electronics| 367| 2.5%  
> Qualcomm| 319| 2.2%  
> Arm| 301| 2.1%  
> Linutronix| 296| 2.0%  
> Oracle| 286| 2.0%  
> IBM| 282| 1.9%  
> Microsoft| 259| 1.8%  
> (Consultant)| 180| 1.2%  
> NXP Semiconductors| 179| 1.2%  
> | By lines changed  
> ---  
> AMD| 125923| 14.9%  
> (Unknown)| 97908| 11.5%  
> Intel| 94150| 11.1%  
> Google| 67461| 8.0%  
> IBM| 48682| 5.7%  
> (None)| 45049| 5.3%  
> Red Hat| 43981| 5.2%  
> Qualcomm| 34014| 4.0%  
> Meta| 26182| 3.1%  
> Microsoft| 19431| 2.3%  
> Linaro| 16389| 1.9%  
> NVIDIA| 16191| 1.9%  
> SUSE| 15175| 1.8%  
> Huawei Technologies| 14136| 1.7%  
> Xilinx| 12961| 1.5%  
> Collabora| 11640| 1.4%  
> Arm| 9357| 1.1%  
> NXP Semiconductors| 8857| 1.0%  
> Rockchip| 8085| 1.0%  
> BayLibre| 8037| 0.9%  
  
This is mostly the usual list of companies that consistently support kernel work from one year to the next. Linutronix has moved up the list this time around, mostly as the result of a lot of work on the kernel's timer subsystem. IBM, once one of the top contributors to the kernel, continues to move downward. 

A different view of how the process works can be had by looking at the Signed-off-by tags applied to patches, specifically those applied by developers other than the author. Those additional signoffs are the traces left when developers forward a patch or apply it to a Git repository on its way toward the mainline; they thus give a clue as to who is doing the work of herding patches upstream. For 6.15, the signoff statistics look like this: 

> Non-author Signed-off-by tags in 6.15  
> ---  
> | Developers  
> ---  
> Jakub Kicinski | 955| 7.0%  
> Mark Brown | 774| 5.7%  
> Andrew Morton| 649| 4.8%  
> Alex Deucher | 571| 4.2%  
> Ingo Molnar | 400| 2.9%  
> Greg Kroah-Hartman | 389| 2.9%  
> Jens Axboe | 325| 2.4%  
> Paolo Abeni | 314| 2.3%  
> Hans Verkuil | 257| 1.9%  
> Thomas Gleixner | 235| 1.7%  
> Christian Brauner | 218| 1.6%  
> Namhyung Kim | 194| 1.4%  
> Jonathan Cameron | 186| 1.4%  
> Alexei Starovoitov | 183| 1.3%  
> Johannes Berg | 160| 1.2%  
> Heiko Stuebner | 148| 1.1%  
> Martin K. Petersen | 137| 1.0%  
> Vinod Koul | 137| 1.0%  
> David Sterba | 131| 1.0%  
> Shawn Guo | 130| 1.0%  
> | Employers  
> ---  
> Meta| 1702| 12.5%  
> Google| 1405| 10.3%  
> Intel| 1310| 9.6%  
> Red Hat| 1151| 8.5%  
> Arm| 955| 7.0%  
> AMD| 908| 6.7%  
> Linaro| 768| 5.7%  
> Microsoft| 427| 3.1%  
> Linux Foundation| 418| 3.1%  
> SUSE| 404| 3.0%  
> (Unknown)| 376| 2.8%  
> (None)| 331| 2.4%  
> Qualcomm| 307| 2.3%  
> NVIDIA| 304| 2.2%  
> Huawei Technologies| 289| 2.1%  
> Linutronix| 283| 2.1%  
> Cisco| 281| 2.1%  
> Oracle| 202| 1.5%  
> LG Electronics| 194| 1.4%  
> IBM| 173| 1.3%  
  
One patch out of every eight going into the kernel now passes through the hands of a maintainer at Meta, and nearly as many are handled by Google developers. 

As of this writing, there are well over 12,000 commits in linux-next, almost all of which can be expected to find their way into the kernel during the 6.16 merge window. That suggests that the next development cycle will be as busy as this one was. As always, keep an eye on LWN to keep up with the next kernel as it is assembled and stabilized.
