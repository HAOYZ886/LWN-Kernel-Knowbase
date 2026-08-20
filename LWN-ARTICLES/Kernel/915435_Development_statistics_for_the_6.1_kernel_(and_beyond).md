---
title: Development statistics for the 6.1 kernel (and beyond)
url: https://lwn.net/Articles/915435/
date: "December 12, 2022"
category: "Releases-6.1"
author: "By Jonathan Corbet December 12, 2022"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 12, 2022

The 6.1 kernel was [released](<https://lwn.net/ml/linux-kernel/CAHk-%3Dwj_HcgFZNyZHTLJ7qC2613zphKDtLh6ndciwopZRfH0aQ%40mail.gmail.com/>) on December 11; by the time of this release, 13,942 non-merge changesets had been pulled into the mainline, growing the kernel by 412,000 lines of code. This is thus not the busiest development cycle ever, but neither is it the slowest, and those changesets contained a number of fundamental changes. This release will also be the long-term-support kernel for 2022. Read on for a look at where the work in 6.1 came from. 

The work in 6.1 was contributed by 2,043 developers, of whom 303 made their first contribution to the kernel in this release. The most active 6.1 developers were: 

> Most active 6.1 developers  
> ---  
> | By changesets  
> ---  
> Krzysztof Kozlowski | 221| 1.6%  
> Yang Yingliang | 169| 1.2%  
> Andy Shevchenko | 145| 1.0%  
> Johan Hovold | 145| 1.0%  
> Zhengchao Shao | 142| 1.0%  
> Maxime Ripard | 127| 0.9%  
> Hans de Goede | 121| 0.9%  
> Vladimir Oltean | 92| 0.7%  
> Jani Nikula | 90| 0.6%  
> Wolfram Sang | 89| 0.6%  
> Dmitry Baryshkov | 87| 0.6%  
> Christoph Hellwig | 85| 0.6%  
> Matthew Wilcox | 84| 0.6%  
> Gaosheng Cui | 81| 0.6%  
> Michael Straube | 73| 0.5%  
> Mark Brown | 71| 0.5%  
> Takashi Iwai | 69| 0.5%  
> Josef Bacik | 68| 0.5%  
> Pavel Begunkov | 67| 0.5%  
> Johannes Berg | 67| 0.5%  
> | By changed lines  
> ---  
> Liam Howlett | 86771| 11.3%  
> Frank Min | 56729| 7.4%  
> Ping-Ke Shih | 21352| 2.8%  
> Ian Rogers | 12588| 1.6%  
> Stephen Hemminger | 12154| 1.6%  
> Miguel Ojeda | 12019| 1.6%  
> Zhengchao Shao | 10771| 1.4%  
> Zong-Zhe Yang | 8101| 1.1%  
> Uwe Kleine-König | 6776| 0.9%  
> Rodrigo Siqueira | 6464| 0.8%  
> Dmitry Baryshkov | 6064| 0.8%  
> Thomas Zimmermann | 5831| 0.8%  
> Alex Elder | 5767| 0.8%  
> Vladimir Oltean | 5740| 0.7%  
> Moudy Ho | 5462| 0.7%  
> Hui.Liu | 5451| 0.7%  
> Sreekanth Reddy | 5263| 0.7%  
> Konrad Dybcio | 5038| 0.7%  
> Geetha sowjanya | 4916| 0.6%  
> Ville Syrjälä | 4899| 0.6%  
  
Krzysztof Kozlowski contributed more changesets to 6.1 than any other developer; this work consisted almost entirely of devicetree changes. Yang Yingliang contributed a lot of cleanup work in the device-driver subsystem. Andy Shevchenko made a wide-ranging set of improvements to various drivers, Johan Hovold contributed driver fixes and devicetree changes, and Zhengchao Shao was active in the networking subsystem. 

The "changed lines" column looks rather different. Liam Howlett only contributed ~~12~~ 59 patches to 6.1, but some of them were big: they added the [maple tree data structure](<https://lwn.net/Articles/845507/>) and the initial uses of it to the memory-management subsystem. Frank Min's three patches added the inevitable set of amdgpu register definitions. Ping-Ke Shih worked on the Realtek rtw89 wireless network adapter, Ian Rogers added a set of Intel event definitions to the perf subsystem, and Stephen Hemminger removed support for the DECnet protocol. 

The top testers and reviewers this time around were: 

> Test and review credits in 6.1  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 200| 13.0%  
> Philipp Hortmann | 134| 8.7%  
> Yu Zhao | 70| 4.6%  
> Gurucharan G | 31| 2.0%  
> Nathan Chancellor | 29| 1.9%  
> Marek Szyprowski | 27| 1.8%  
> Victor Nogueira | 26| 1.7%  
> Linux Kernel Functional Testing | 26| 1.7%  
> Naresh Kamboju | 25| 1.6%  
> Peter Zijlstra | 22| 1.4%  
> Kees Cook | 21| 1.4%  
> Shaopeng Tan | 21| 1.4%  
> Alexander Stein | 21| 1.4%  
> Xin Hao | 21| 1.4%  
> Cristian Marussi | 21| 1.4%  
> | Reviewed-by  
> ---  
> Andy Shevchenko | 229| 3.0%  
> Krzysztof Kozlowski | 211| 2.8%  
> Hans de Goede | 147| 1.9%  
> David Sterba | 131| 1.7%  
> AngeloGioacchino Del Regno | 131| 1.7%  
> Rob Herring | 130| 1.7%  
> Dmitry Baryshkov | 114| 1.5%  
> Kees Cook | 91| 1.2%  
> Tariq Toukan | 89| 1.2%  
> Hawking Zhang | 83| 1.1%  
> Linus Walleij | 67| 0.9%  
> Laurent Pinchart | 67| 0.9%  
> Jan Kara | 67| 0.9%  
> Guenter Roeck | 66| 0.9%  
> Andrew Lunn | 66| 0.9%  
  
Daniel Wheeler's presence at the top of the test-credit column is pretty much a given at this point; he continues to test patches from his colleagues at AMD at a rate of two or three patches every day of the development cycle. Philipp Hortmann, instead, tested patches to Realtek drivers in the staging tree. Yu Zhao's test credits appear in various memory-management changes, mostly related to the maple-tree work. On the review side, Andy Shevchenko reviewed patches for work all over the kernel tree, while Krzysztof Kozlowski focused mostly on device-tree changes and Hans de Goede reviewed a lot of platform-driver patches. 

The most active employers this time around were: 

> Most active 6.1 employers  
> ---  
> | By changesets  
> ---  
> Huawei Technologies| 1281| 9.2%  
> Intel| 1254| 9.0%  
> (Unknown)| 1097| 7.9%  
> Google| 917| 6.6%  
> Linaro| 837| 6.0%  
> AMD| 750| 5.4%  
> Red Hat| 672| 4.8%  
> (None)| 564| 4.0%  
> Meta| 414| 3.0%  
> NVIDIA| 389| 2.8%  
> SUSE| 333| 2.4%  
> Oracle| 318| 2.3%  
> NXP Semiconductors| 275| 2.0%  
> IBM| 260| 1.9%  
> Renesas Electronics| 224| 1.6%  
> (Consultant)| 208| 1.5%  
> Microchip Technology Inc.| 192| 1.4%  
> Arm| 187| 1.3%  
> MediaTek| 164| 1.2%  
> Collabora| 144| 1.0%  
> | By lines changed  
> ---  
> Oracle| 91852| 12.0%  
> AMD| 89761| 11.7%  
> Google| 56504| 7.4%  
> Intel| 44062| 5.8%  
> (Unknown)| 33765| 4.4%  
> Realtek| 33277| 4.3%  
> Linaro| 31234| 4.1%  
> Huawei Technologies| 27856| 3.6%  
> NVIDIA| 25441| 3.3%  
> Red Hat| 24073| 3.1%  
> (None)| 21498| 2.8%  
> Meta| 18783| 2.5%  
> MediaTek| 17599| 2.3%  
> NXP Semiconductors| 14342| 1.9%  
> SUSE| 13749| 1.8%  
> Brocade| 12154| 1.6%  
> Microchip Technology Inc.| 11651| 1.5%  
> Pengutronix| 10200| 1.3%  
> Broadcom| 8054| 1.1%  
> Marvell| 8036| 1.0%  
  
Huawei clearly had a busy development cycle, with 117 developers contributing changes throughout the kernel. Otherwise, these results show yet another fairly typical development cycle. 

#### Looking back

The kernel's development cycle runs for nine or ten weeks before producing the next major release. There is, however, another cycle built on top of that. The last release from each calendar year receives long-term support, for a period of up to six years. These kernels are the ones that end up in most products and distributions over time; at this point, one could maybe say that the LTS cycle is the real kernel release cycle, with the other releases just being intermediate stabilization points. 

Unless something extremely surprising happens, 6.1 will be the final kernel release for 2022, and thus will become the next LTS kernel. Given that 6.1 is the endpoint for the year-long LTS development cycle, then a look at the full cycle is also potentially of interest. The previous LTS kernel, 5.15, was released on October 31 2021. Since then, the kernel community has merged 86,660 patches from 5,034 developers — 1,741 of whom were first-time contributors — with a net growth of over 3.7 million lines of code. The most active developers over this entire period were: 

> Most active developers, 5.16 through 6.1  
> ---  
> | By changesets  
> ---  
> Krzysztof Kozlowski | 1134| 1.3%  
> Christoph Hellwig | 918| 1.1%  
> Matthew Wilcox | 716| 0.8%  
> Sean Christopherson | 687| 0.8%  
> Andy Shevchenko | 683| 0.8%  
> Ville Syrjälä | 646| 0.7%  
> Michael Straube | 631| 0.7%  
> Jakub Kicinski | 583| 0.7%  
> Geert Uytterhoeven | 560| 0.6%  
> Martin Kaiser | 552| 0.6%  
> Hans de Goede | 536| 0.6%  
> Dmitry Baryshkov | 516| 0.6%  
> Jani Nikula | 487| 0.6%  
> Mark Brown | 471| 0.5%  
> Vladimir Oltean | 466| 0.5%  
> Christophe JAILLET | 454| 0.5%  
> Johannes Berg | 453| 0.5%  
> Eric Dumazet | 447| 0.5%  
> Pavel Begunkov | 445| 0.5%  
> Mauro Carvalho Chehab | 430| 0.5%  
> | By changed lines  
> ---  
> Aurabindo Pillai | 341685| 5.5%  
> Leo Li | 227954| 3.7%  
> Hawking Zhang | 225057| 3.7%  
> Qingqing Zhuo | 198735| 3.2%  
> Huang Rui | 197305| 3.2%  
> Roman Li | 155944| 2.5%  
> Zhengjun Xing | 152525| 2.5%  
> Oded Gabbay | 150670| 2.4%  
> Ping-Ke Shih | 147214| 2.4%  
> Ian Rogers | 145313| 2.4%  
> Dmitry Baryshkov | 92702| 1.5%  
> Liam Howlett | 86859| 1.4%  
> Jakub Kicinski | 81325| 1.3%  
> Frank Min | 56729| 0.9%  
> Christoph Hellwig | 46400| 0.8%  
> Martin Habets | 44438| 0.7%  
> Zhan Liu | 34647| 0.6%  
> David Howells | 31466| 0.5%  
> Krzysztof Kozlowski | 30380| 0.5%  
> Nick Terrell | 28611| 0.5%  
  
The most prolific developer during this time, once again Krzysztof Kozlowski, contributed over three patches for every single day of this extended development cycle — and still accounted for only 1.3% of the total. The top _seven_ contributors in the "lines changed" column got there as the result of adding amdgpu register definitions; they account for well over 1 million lines of added code, which is a large portion of the total growth of the kernel. 

That effect can be seen in the employer numbers as well: 

> Most active employers, 5.16 through 6.1  
> ---  
> | By changesets  
> ---  
> Intel| 9295| 10.7%  
> (Unknown)| 6134| 7.1%  
> Google| 5597| 6.5%  
> Red Hat| 4916| 5.7%  
> AMD| 4474| 5.2%  
> Linaro| 4373| 5.0%  
> (None)| 4029| 4.6%  
> Huawei Technologies| 3649| 4.2%  
> Meta| 2904| 3.4%  
> NVIDIA| 2563| 3.0%  
> SUSE| 2269| 2.6%  
> (Consultant)| 2018| 2.3%  
> Oracle| 1944| 2.2%  
> IBM| 1703| 2.0%  
> Renesas Electronics| 1487| 1.7%  
> Arm| 1391| 1.6%  
> NXP Semiconductors| 1292| 1.5%  
> MediaTek| 1212| 1.4%  
> Alibaba| 944| 1.1%  
> Microchip Technology Inc.| 943| 1.1%  
> | By lines changed  
> ---  
> AMD| 1653237| 26.8%  
> Intel| 744059| 12.1%  
> Google| 345341| 5.6%  
> Linaro| 302581| 4.9%  
> (Unknown)| 212408| 3.4%  
> Meta| 209040| 3.4%  
> Red Hat| 202698| 3.3%  
> Realtek| 185932| 3.0%  
> NVIDIA| 179426| 2.9%  
> (None)| 165999| 2.7%  
> Oracle| 132210| 2.1%  
> MediaTek| 112561| 1.8%  
> Huawei Technologies| 101436| 1.6%  
> (Consultant)| 86227| 1.4%  
> SUSE| 81624| 1.3%  
> NXP Semiconductors| 71782| 1.2%  
> Xilinx| 66447| 1.1%  
> IBM| 65059| 1.1%  
> Renesas Electronics| 58418| 0.9%  
> Microchip Technology Inc.| 50529| 0.8%  
  
The employer numbers do not change much from one cycle to the next, so it is unsurprising that a year's worth of numbers looks about the same as well. AMD's showing in the "lines changed" column demonstrates what the results of regularly dumping machine-generated register definitions into the kernel can be. 

Finally, another metric of interest is non-author signoffs: the application of a Signed-off-by tag to a patch written by somebody else. That normally happens when a maintainer accepts a patch and adds it to their repository to eventually send upstream. These tags can thus reveal who is doing the maintainer work in the kernel community: 

> Non-author signoffs, 5.16 through 6.1  
> ---  
> | Maintainers  
> ---  
> Greg Kroah-Hartman | 5242| 6.3%  
> David S. Miller | 4605| 5.6%  
> Mark Brown | 3708| 4.5%  
> Alex Deucher | 3689| 4.4%  
> Jakub Kicinski | 2968| 3.6%  
> Andrew Morton| 2900| 3.5%  
> Jens Axboe | 1925| 2.3%  
> Mauro Carvalho Chehab | 1872| 2.3%  
> Bjorn Andersson | 1729| 2.1%  
> Martin K. Petersen | 1490| 1.8%  
> Paolo Bonzini | 1456| 1.8%  
> Kalle Valo | 1290| 1.6%  
> Arnaldo Carvalho de Melo | 1221| 1.5%  
> Michael Ellerman | 1109| 1.3%  
> Vinod Koul | 1023| 1.2%  
> David Sterba | 986| 1.2%  
> Alexei Starovoitov | 957| 1.2%  
> Jonathan Cameron | 948| 1.1%  
> Hans Verkuil | 944| 1.1%  
> Shawn Guo | 888| 1.1%  
> | Employers  
> ---  
> Red Hat| 10945| 13.2%  
> Linaro| 10488| 12.6%  
> Intel| 8115| 9.8%  
> Meta| 6946| 8.4%  
> Google| 6649| 8.0%  
> Linux Foundation| 5738| 6.9%  
> AMD| 4058| 4.9%  
> SUSE| 3526| 4.3%  
> NVIDIA| 2315| 2.8%  
> Huawei Technologies| 2070| 2.5%  
> IBM| 1916| 2.3%  
> (Consultant)| 1887| 2.3%  
> Oracle| 1854| 2.2%  
> Qualcomm| 1696| 2.0%  
> (None)| 1533| 1.8%  
> Arm| 1270| 1.5%  
> (Unknown)| 1094| 1.3%  
> Cisco| 950| 1.1%  
> Microsoft| 580| 0.7%  
> Renesas Electronics| 552| 0.7%  
  
The most active maintainers deal with dozens of patches every day and somehow manage to keep their sanity anyway. The list of companies employing maintainers has changed a bit over time; Linaro has been moving up for some time, for example. But it remains true that relatively few companies support the maintainer role; over half of the patches being merged into the mainline kernel pass through the hands of developers working for just five companies. One of the best ways for companies to improve their support for kernel development would be to give their developers the time and encouragement to become maintainers. 

All told, though, the kernel's development process continues to move forward at a rapid pace, producing releases on a regular schedule and bringing several new developers into the community every day. The overall picture shows a community that is seemingly in good health and continuing to manage the challenges posed by its fast pace of development.
