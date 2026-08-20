---
title: Development statistics for 6.19
url: https://lwn.net/Articles/1057302/
date: "February 9, 2026"
category: "Releases-6.19"
author: "By Jonathan Corbet February 9, 2026"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 9, 2026

Linus Torvalds [released the 6.19 kernel](<https://lwn.net/ml/all/CAHk-=wh0Fj7yE7iuW8awFCFt53s9T186qNbZX673E+oNCeQSFg@mail.gmail.com/>) on February 8, as expected. This development cycle brought 14,344 non-merge changesets into the mainline, making it the busiest release since 6.16 in July 2025. As usual, we have put together a set of statistics on where these changes come from, along with a quick look at how long new kernel developers stay around. 

As a reminder: LWN subscribers can find much of the information below — and more — at any time in the [LWN kernel source database](</ksdb/>). 

The 6.19 development cycle brought in the work from 2,141 developers, which just barely beats the previous record (2,134) set for 6.18; 333 of those developers made their first contribution to the kernel in 6.19, also a relatively high number. The most active developers for 6.19 were: 

> Most active 6.19 developers  
> ---  
> | By changesets  
> ---  
> Kuninori Morimoto | 459| 3.2%  
> Christian Brauner | 271| 1.9%  
> Johan Hovold | 158| 1.1%  
> Ville Syrjälä | 153| 1.1%  
> Ian Rogers | 140| 1.0%  
> Russell King | 124| 0.9%  
> Josh Poimboeuf | 101| 0.7%  
> Andy Shevchenko | 100| 0.7%  
> Krzysztof Kozlowski | 93| 0.6%  
> Jani Nikula | 91| 0.6%  
> Sean Christopherson | 88| 0.6%  
> Filipe Manana | 87| 0.6%  
> Marco Crivellari | 87| 0.6%  
> Christoph Hellwig | 86| 0.6%  
> Thomas Zimmermann | 85| 0.6%  
> Eric Dumazet | 85| 0.6%  
> Peter Zijlstra| 82| 0.6%  
> Marc Zyngier | 82| 0.6%  
> Frank Li | 78| 0.5%  
> SeongJae Park | 78| 0.5%  
> | By changed lines  
> ---  
> Miguel Ojeda | 58000| 7.8%  
> Cyril Chao | 19755| 2.6%  
> Christian Brauner | 16604| 2.2%  
> YiPeng Chai | 13293| 1.8%  
> Dmitry Baryshkov | 12244| 1.6%  
> Ian Rogers | 10933| 1.5%  
> Jason Gunthorpe | 10851| 1.5%  
> Eric Biggers | 9549| 1.3%  
> Daniel Scally | 9429| 1.3%  
> AngeloGioacchino Del Regno | 6201| 0.8%  
> Josh Poimboeuf | 6010| 0.8%  
> Ilya Bakoulin | 6009| 0.8%  
> Rob Herring| 5777| 0.8%  
> Johannes Berg | 5707| 0.8%  
> Svyatoslav Ryhel | 5610| 0.8%  
> Akhil P Oommen | 5516| 0.7%  
> Mauro Carvalho Chehab | 5196| 0.7%  
> Neilay Kharwadkar | 5162| 0.7%  
> Igor Belwon | 5155| 0.7%  
> Lorenzo Stoakes | 4830| 0.6%  
  
Kuninori Morimoto, who was first seen during the 2.6.28 development cycle in 2008, was the biggest contributor of changesets by virtue of a major refactoring effort in the sound subsystem. Christian Brauner, the maintainer of the virtual filesystem layer, refactored the handling of credentials, added [the `listns()` system call](<https://lwn.net/Articles/1043824/>), and added many self tests, among other contributions. Johan Hovold fixed numerous bugs and did a lot of cleanups in various driver subsystems. Ville Syrjälä worked extensively in the i915 graphics-driver subsystem, and Ian Rogers contributed a long list of improvements to the `perf` tool. 

Looking at lines changed, Miguel Ojeda topped the list with [the addition of a modified version](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=808c999fc9e7>) of [the Rust `syn` crate](<https://docs.rs/syn/latest/syn/>). Cyril Chao's first-ever kernel contribution, which put him into second place in the "lines-changed" list, was a driver for MediaTek mt8189 platform devices. YiPeng Chai worked with the amdgpu graphics driver, and Dmitry Baryshkov updated devicetree files for a number of Qualcomm devices. 

A full 10% of the patches merged for 6.19 had Tested-by tags, while 56% had Reviewed-by tags; both of those numbers are slightly higher than usual. The top testers and reviews for this release were: 

> Test and review credits in 6.19  
> ---  
> | Tested-by  
> ---  
> Dan Wheeler | 89| 4.8%  
> Joe Lawrence | 63| 3.4%  
> Mark Brown | 55| 3.0%  
> Randy Dunlap | 53| 2.9%  
> Fuad Tabba | 53| 2.9%  
> Lad Prabhakar | 37| 2.0%  
> Shaopeng Tan | 35| 1.9%  
> James Clark | 34| 1.8%  
> Carl Worth | 34| 1.8%  
> Hanjun Guo | 33| 1.8%  
> Gavin Shan | 32| 1.7%  
> Zeng Heng | 32| 1.7%  
> Ryan Walklin | 30| 1.6%  
> Kai Huang | 29| 1.6%  
> Fenghua Yu | 28| 1.5%  
> Yan Zhao | 28| 1.5%  
> | Reviewed-by  
> ---  
> Charles Keepax | 310| 2.8%  
> Dmitry Baryshkov | 191| 1.7%  
> Geert Uytterhoeven | 164| 1.5%  
> Frank Li | 158| 1.4%  
> Krzysztof Kozlowski | 156| 1.4%  
> David Sterba | 144| 1.3%  
> Christoph Hellwig | 139| 1.3%  
> Konrad Dybcio | 125| 1.1%  
> Simon Horman | 125| 1.1%  
> Ilpo Järvinen | 118| 1.1%  
> Jan Kara | 118| 1.1%  
> Jeff Layton | 113| 1.0%  
> AngeloGioacchino Del Regno | 111| 1.0%  
> Jonathan Cameron | 109| 1.0%  
> Andrew Lunn | 109| 1.0%  
> Ville Syrjälä | 105| 1.0%  
  
The list of top reviewers is a bit different than in the past; somehow Charles Keepax managed to review 310 commits — more than four for every day of this 70-day release cycle — mostly within the sound-driver subsystem. The other top reviewers were focused on system-on-chip drivers and devicetree-related changes. The list of top testers is more typical, with Daniel Wheeler on top as usual. 

The development of the 6.19 kernel was supported by 227 employers that we know of. The most active employers were: 

> Most active 6.19 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1591| 11.1%  
> (Unknown)| 1410| 9.8%  
> Google| 1099| 7.7%  
> Red Hat| 829| 5.8%  
> Renesas Electronics| 741| 5.2%  
> AMD| 612| 4.3%  
> (None)| 554| 3.9%  
> Qualcomm| 485| 3.4%  
> SUSE| 462| 3.2%  
> Microsoft| 434| 3.0%  
> NVIDIA| 407| 2.8%  
> (Consultant)| 392| 2.7%  
> Meta| 379| 2.6%  
> Oracle| 371| 2.6%  
> NXP Semiconductors| 325| 2.3%  
> Linaro| 319| 2.2%  
> Huawei Technologies| 260| 1.8%  
> IBM| 236| 1.6%  
> Arm| 200| 1.4%  
> Bootlin| 160| 1.1%  
> | By lines changed  
> ---  
> Google| 101728| 13.6%  
> (Unknown)| 70125| 9.4%  
> Intel| 59934| 8.0%  
> AMD| 45025| 6.0%  
> Qualcomm| 36322| 4.9%  
> NVIDIA| 32585| 4.4%  
> Red Hat| 31429| 4.2%  
> Microsoft| 30621| 4.1%  
> (None)| 27285| 3.7%  
> MediaTek| 23223| 3.1%  
> Ideas on Board| 14880| 2.0%  
> Renesas Electronics| 14350| 1.9%  
> Meta| 14232| 1.9%  
> Collabora| 14088| 1.9%  
> Huawei Technologies| 13257| 1.8%  
> SUSE| 13209| 1.8%  
> Oracle| 12943| 1.7%  
> Arm| 12761| 1.7%  
> IBM| 12230| 1.6%  
> Linaro| 9015| 1.2%  
  
These numbers are reasonably consistent with recent history; hardware vendors are still contributing a large share of the changes. When considering which companies are most influential in kernel development, though, one should also look at the Signed-off-by tags added to patches by maintainers as they apply those patches to their repositories: 

> Non-author signoffs in 6.19  
> ---  
> | Individual  
> ---  
> Jakub Kicinski | 991| 7.5%  
> Mark Brown | 946| 7.2%  
> Andrew Morton| 584| 4.4%  
> Alex Deucher | 478| 3.6%  
> Greg Kroah-Hartman | 406| 3.1%  
> Jens Axboe | 277| 2.1%  
> Hans Verkuil | 257| 2.0%  
> Bjorn Andersson | 245| 1.9%  
> Paolo Abeni | 218| 1.7%  
> Christian Brauner | 200| 1.5%  
> Namhyung Kim | 196| 1.5%  
> Shawn Guo | 185| 1.4%  
> Martin K. Petersen | 184| 1.4%  
> Peter Zijlstra | 184| 1.4%  
> David Sterba | 179| 1.4%  
> Jonathan Cameron | 167| 1.3%  
> Alexei Starovoitov | 141| 1.1%  
> Geert Uytterhoeven | 129| 1.0%  
> Jonathan Corbet | 121| 0.9%  
> Ilpo Järvinen | 121| 0.9%  
> | By employer  
> ---  
> Meta| 1544| 11.8%  
> Google| 1296| 9.9%  
> Intel| 1243| 9.5%  
> Arm| 1171| 8.9%  
> AMD| 848| 6.5%  
> Linaro| 786| 6.0%  
> Red Hat| 647| 4.9%  
> Qualcomm| 564| 4.3%  
> Linux Foundation| 441| 3.4%  
> Microsoft| 432| 3.3%  
> SUSE| 423| 3.2%  
> (Unknown)| 409| 3.1%  
> NVIDIA| 312| 2.4%  
> Cisco| 257| 2.0%  
> Oracle| 235| 1.8%  
> Huawei Technologies| 233| 1.8%  
> (None)| 209| 1.6%  
> LG Electronics| 196| 1.5%  
> Renesas Electronics| 174| 1.3%  
> IBM| 133| 1.0%  
  
The top two companies here are both of the hyperscaler variety, with the top being Meta, which appears rather farther down in the list of changeset contributors. While Meta does contribute a lot of patches — and significant core patches at that — it also employs the maintainers that handle a lot more patches authored by others. Arm, too, shows a bigger influence by this metric. 

#### Developer longevity

For as long as the kernel community has existed, people have worried about its ability to attract new developers. I have often pointed out that each release features the work of roughly 300 first-time developers; the response is often to ask how long those developers stay around. The time has come to try to give at least a partial answer to that question. The plot below was generated by accumulating a list of the 5,424 developers who made their first contribution to one of the 5.x mainline kernels, then looking at how many other releases each contributed to. 

> ![\[longevity plot\]](https://static.lwn.net/images/2026/first-time-longevity.svg)

What we see is that 1,943 of those first-time contributors — 36% of the total — were never seen again after contributing to one release. Another 883 developers (16%) showed up for one other release, and so on. In the end, 32% of the first-time contributors during this period have been present for at least four kernel releases. At the long tail of the distribution, there are two first-time developers (Andrii Nakryiko and Vladimir Oltean) who have only missed one release and two (Stephan Gerhold and Stefano Garzarella) who have contributed to every release from 5.0 to 6.19. 

To provide one other small data point: the first-time contributors being studied here arrived between early 2019 (the 5.0 release) and mid-2022 (5.19). Of those 5,424 developers, 1,067 (just under 20%) of them contributed to at least one of the releases made in 2025. It seems reasonable to consider that group as still being active in the kernel community. Whether these results are good or bad is probably a matter for debate, but it does seem clear that, while a lot of contributors pass through quickly, others are staying around for the long haul. 

The kernel community as a whole is also clearly here for the long haul; the process shows no real signs of slowing down. As of this writing, there are just short of 11,000 changesets in the linux-next repository; most of those will move into the mainline in the upcoming merge window. The next kernel, which will be called 7.0, will continue to demonstrate the community's fast pace; stay tuned for the details.
