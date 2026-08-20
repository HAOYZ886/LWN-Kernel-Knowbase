---
title: Development statistics for the 6.5 kernel
url: https://lwn.net/Articles/941675/
date: "August 28, 2023"
category: "Releases-6.5"
author: "By Jonathan Corbet August 28, 2023"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 28, 2023

The 6.5 kernel was [released](<https://lwn.net/ml/linux-kernel/CAHk-=wgmKhCrdrOCjp=5v9NO6C=PJ8ZTZcCXj09piHzsZ7qqmw@mail.gmail.com/>) on August 27 after a nine-week development cycle. By that time, some 13,561 non-merge changesets had found their way into the mainline repository, the lowest number seen since the 5.15 release (12,377 changesets) in late 2021. Nonetheless, quite a bit of significant work was done in this cycle; read on for a look at where that work came from. 

1,921 Developers contributed to 6.5, a slightly lower number than usual; 271 of those developers made their first kernel contribution for this release. The top contributors to the 6.5 kernel were: 

> Most active 6.5 developers  
> ---  
> | By changesets  
> ---  
> Uwe Kleine-König | 272| 2.0%  
> Krzysztof Kozlowski | 235| 1.7%  
> Christoph Hellwig | 213| 1.6%  
> Ian Rogers | 198| 1.5%  
> Arnd Bergmann | 190| 1.4%  
> Hans de Goede | 140| 1.0%  
> David Howells | 120| 0.9%  
> Lijo Lazar | 115| 0.8%  
> Johannes Berg | 110| 0.8%  
> Dmitry Baryshkov | 108| 0.8%  
> Konrad Dybcio | 99| 0.7%  
> Mark Brown | 96| 0.7%  
> Jani Nikula | 96| 0.7%  
> Thomas Gleixner | 91| 0.7%  
> Oswald Buddenhagen | 91| 0.7%  
> Christophe JAILLET | 88| 0.6%  
> Nishanth Menon | 87| 0.6%  
> Andy Shevchenko | 85| 0.6%  
> Jakub Kicinski | 84| 0.6%  
> Thomas Zimmermann | 74| 0.5%  
> | By changed lines  
> ---  
> Ian Rogers | 26662| 4.3%  
> Zong-Zhe Yang | 20395| 3.3%  
> Jakub Kicinski | 17505| 2.8%  
> Hawking Zhang | 16730| 2.7%  
> Mark Rutland | 14829| 2.4%  
> Ping-Ke Shih | 11090| 1.8%  
> Rohit Agarwal | 8669| 1.4%  
> Benjamin Gaignard | 8275| 1.3%  
> Takashi Iwai | 7675| 1.2%  
> Hans de Goede | 6112| 1.0%  
> Jonathan Kim | 5820| 0.9%  
> Le Ma | 5337| 0.9%  
> Jani Nikula | 5334| 0.9%  
> Konrad Dybcio | 5263| 0.8%  
> Bryan O'Donoghue | 5119| 0.8%  
> Christoph Hellwig | 4924| 0.8%  
> David Howells | 4848| 0.8%  
> Pawel Laszczak | 4827| 0.8%  
> Dmitry Baryshkov | 4773| 0.8%  
> Johannes Berg | 4748| 0.8%  
  
As was the case for 6.4, Uwe Kleine-König tops the list of individual contributors as measured by number of changesets, again mostly for work changing the internal platform-device driver API. Krzysztof Kozlowski is, again as with 6.4, in the second spot, having contributed a lot of devicetree changes. Christoph Hellwig continues refactoring code in the block and filesystem layers, with much of his work being in the Btrfs filesystem this time around. Ian Rogers (again, as with 6.4) contributed a lot of changes to the `perf` tool, and Arnd Bergmann made fixes all over the kernel tree. 

Rogers touched the most lines of code this time around (in a cycle that featured relatively few large commits) by updating various vendor-specific perf events. Zong-Zhe Yang contributed 25 changes to the rtw89 WiFi driver, Jakub Kicinski added code to the [ynl tool](<https://www.kernel.org/doc/html/latest/userspace-api/netlink/intro-specs.html>), Hawking Zhang added a relatively restrained set of amdgpu headers, and Mark Rutland reorganized and documented a number of low-level locking primitives. 

The top testers and reviewers for this cycle were: 

> Test and review credits in 6.5  
> ---  
> | Tested-by  
> ---  
> Daniel Wheeler | 165| 11.5%  
> Michael Kelley | 63| 4.4%  
> Kan Liang | 44| 3.1%  
> Guilherme G. Piccoli | 40| 2.8%  
> Helge Deller | 38| 2.6%  
> Oleksandr Natalenko | 37| 2.6%  
> Heiko Stuebner | 32| 2.2%  
> Randy Dunlap | 25| 1.7%  
> Naama Meir | 21| 1.5%  
> Rafal Romanowski | 19| 1.3%  
> Pucha Himasekhar Reddy | 19| 1.3%  
> | Reviewed-by  
> ---  
> Simon Horman | 276| 3.1%  
> Konrad Dybcio | 220| 2.5%  
> Christoph Hellwig | 218| 2.5%  
> Hawking Zhang | 161| 1.8%  
> Kees Cook | 153| 1.7%  
> Andy Shevchenko | 147| 1.7%  
> David Sterba | 142| 1.6%  
> Krzysztof Kozlowski | 142| 1.6%  
> Felix Kuehling | 138| 1.6%  
> Conor Dooley | 115| 1.3%  
> Dmitry Baryshkov | 113| 1.3%  
  
There are two developers who are credited with having tested at least one patch for every day in this 63-day cycle, and 25 who reviewed at least one patch per day. Simon Horman, who reviews patches in the network subsystem, sustained a rate of over four per day. Overall, 1,177 (8.7%) of the commits in 6.5 had Tested-by tags, and 6,702 (49%) had Reviewed-by tags, continuing the longstanding, gradual increase in the use of those tags. 

A total of 218 employers (that we were able to identify) supported work on the 6.5 kernel; the most active of those were: 

> Most active 6.5 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1503| 11.1%  
> AMD| 1233| 9.1%  
> Linaro| 1174| 8.7%  
> Google| 922| 6.8%  
> (Unknown)| 838| 6.2%  
> Red Hat| 777| 5.7%  
> (None)| 623| 4.6%  
> NVIDIA| 381| 2.8%  
> SUSE| 366| 2.7%  
> Huawei Technologies| 347| 2.6%  
> Pengutronix| 326| 2.4%  
> Qualcomm| 303| 2.2%  
> Oracle| 293| 2.2%  
> Meta| 265| 2.0%  
> (Consultant)| 261| 1.9%  
> IBM| 236| 1.7%  
> Texas Instruments| 193| 1.4%  
> Arm| 180| 1.3%  
> Renesas Electronics| 151| 1.1%  
> Collabora| 147| 1.1%  
> | By lines changed  
> ---  
> Intel| 66098| 10.7%  
> AMD| 65508| 10.6%  
> Google| 46208| 7.4%  
> Linaro| 41969| 6.8%  
> Realtek| 34719| 5.6%  
> (Unknown)| 31602| 5.1%  
> Red Hat| 26304| 4.2%  
> Qualcomm| 23527| 3.8%  
> Meta| 22721| 3.7%  
> Arm| 18813| 3.0%  
> Collabora| 16801| 2.7%  
> (None)| 16168| 2.6%  
> SUSE| 15021| 2.4%  
> NVIDIA| 12800| 2.1%  
> Texas Instruments| 11308| 1.8%  
> Oracle| 8627| 1.4%  
> Huawei Technologies| 7908| 1.3%  
> IBM| 7311| 1.2%  
> (Consultant)| 6897| 1.1%  
> MediaTek| 6562| 1.1%  
  
The list of companies working on the kernel tends not to change much — at the upper end of the list, anyway. This list says little about what these companies are up to in the kernel, though. Below is a summary, for some of the top corporations, of the number of changesets touching some of the more active areas of the kernel (and the percentage that makes of the company's total contribution): 

> | AMD | Google | Intel | Linaro | NVIDIA | Red Hat | (None) | (Unknown)  
> ---|---|---|---|---|---|---|---|---  
> Architecture | 65| 5% | 213 | 23% | 114| 8% | 347| 30% | 15|  4% | 40| 5% | 111 | 18% | 135| 17%  
> Core kernel | 33| 3% | 182| 20% | 147 |  10% |  74|  6% | 15| 4% | 51| 7% | 75|  12% | 53| 7%  
> Block/filesystem | 0| 0% | 62| 7% | 19 |  1% |  15|  1% | 0|  0% | 154| 21% | 26| 4% | 42| 5%  
> Drivers | 1,106| 91% | 199| 22% | 1064 |  73% | 597 |  52% |  275|  75% | 286| 39% | 366|  60% | 413| 51%  
> Networking | 10| 1% | 74| 8% | 313 |  21% |  60| 5% | 221| 60% | 133|  18% | 79|  13% | 104|  13%  
> Tools | 17| 1% | 235| 26% | 55 |  4% | 17| 1% | 58|  16% | 112| 15% | 5| 1% | 41|  5%  
> Documentation | 27| 2% | 39| 4% | 29 |  2% | 148| 13% | 15| 4% | 31| 4% | 53|  9% | 97|  12%  
  
This table gives some idea of where the top contributors put their effort in the kernel. The sharp-eyed may notice that the percentages do not add up to 100%; that is primarily because many changes touch more than one subsystem, and are thus counted more than once. 

It will not be surprising to see that the hardware-manufacturing companies — AMD, Intel, and NVIDIA — focus their work in the driver subsystem. They are here first and foremost to ensure that their products work well with Linux, and that means working on drivers. Perhaps a bit more surprising is that these companies, all of which make CPUs, do not make many changes to the architecture-support code (or, at least, did not do that during the 6.5 development cycle). Google's commit count under `arch/` exceeded that of those other three companies combined. 

Linaro, which exists to support the Arm architecture, does put a fair amount of effort into the `arch/` subtree. Lest one think that Linaro is a documentation champion, though, one should remember that the devicetree bindings live under `Documentation/`. That work is really more hardware support in the end. Of the groups listed above, developers with no known affiliation (the "None" and "Unknown" columns) are seemingly the most enthusiastic, percentage-wise, about contributing to the documentation. 

A lot of work, from some companies at least, is done in the `tools/` subtree. Activity there is focused on the `perf` tool, the growing self-test suite, the `objtool` utility that is an ever-more important part of the build process, and more. An increasing amount of the code in the kernel tree runs in user space. 

Kernel development goes through periods of high and low activity; 6.5 was a slow cycle (though, clearly, in relative terms), but 6.6 might pick up the pace a bit. There are 11,642 non-merge changesets queued there as of this writing. At times, it seems, there is simply less work needing to be done. Experience says, though, that the patch rate will pick up again before too long; the kernel project, it seems, is never done.
