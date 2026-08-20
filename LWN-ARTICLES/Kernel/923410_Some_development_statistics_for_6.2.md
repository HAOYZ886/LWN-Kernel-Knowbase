---
title: Some development statistics for 6.2
url: https://lwn.net/Articles/923410/
date: "February 20, 2023"
category: "Releases-6.2"
author: "By Jonathan Corbet February 20, 2023"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 20, 2023

The 6.2 kernel was [released](<https://lwn.net/Articles/923744/>) on February 19, at the end of a ten-week development cycle. This time around, 15,536 non-merge changesets found their way into the mainline repository, making this cycle significantly more active than [its predecessor](<https://lwn.net/Articles/915435/>). Read on for a look at the work that went into this kernel release. 

The work in 6.2 was contributed by 2,088 developers, which just barely sets a new record; the previous record was the 2,086 developers contributed to 5.19. Of those developers, 294 made their first contribution to the kernel in this cycle, a fairly typical number. The most active contributors to the 6.2 release were: 

> Most active 6.2 developers  
> ---  
> | By changesets  
> ---  
> Uwe Kleine-König | 571| 3.7%  
> Krzysztof Kozlowski | 342| 2.2%  
> Johan Hovold | 178| 1.1%  
> Andy Shevchenko | 152| 1.0%  
> Thomas Gleixner | 148| 1.0%  
> Ville Syrjälä | 147| 0.9%  
> Yang Yingliang | 141| 0.9%  
> Ben Skeggs | 128| 0.8%  
> Christoph Hellwig | 126| 0.8%  
> Dmitry Baryshkov | 120| 0.8%  
> Namhyung Kim | 117| 0.8%  
> Sean Christopherson | 112| 0.7%  
> Colin Ian King | 103| 0.7%  
> David Howells | 99| 0.6%  
> Martin Kaiser | 97| 0.6%  
> Ian Rogers | 96| 0.6%  
> Josef Bacik | 94| 0.6%  
> Hans de Goede | 93| 0.6%  
> Dmitry Torokhov | 80| 0.5%  
> Thomas Zimmermann | 77| 0.5%  
> | By changed lines  
> ---  
> Ian Rogers | 149899| 17.0%  
> Ping-Ke Shih | 31997| 3.6%  
> Ben Skeggs | 23004| 2.6%  
> Steen Hegelund | 17222| 2.0%  
> Arnd Bergmann | 13042| 1.5%  
> Shayne Chen | 12476| 1.4%  
> Jason Gunthorpe | 11480| 1.3%  
> Krzysztof Kozlowski | 11365| 1.3%  
> Eugen Hristev | 10452| 1.2%  
> David Howells | 9020| 1.0%  
> Dmitry Baryshkov | 7981| 0.9%  
> Daniel Almeida | 7654| 0.9%  
> Nick Terrell | 6984| 0.8%  
> Carlos Bilbao | 6817| 0.8%  
> Paul Kocialkowski | 6487| 0.7%  
> Josef Bacik | 6219| 0.7%  
> Johan Hovold | 5726| 0.6%  
> Kumar Kartikeya Dwivedi | 5615| 0.6%  
> Tianjia Zhang | 5531| 0.6%  
> Horatiu Vultur | 5488| 0.6%  
  
Uwe Kleine-König contributed the most changesets this time around; this work was dominated by the conversion of a vast number of drivers to a new I2C API. Krzysztof Kozlowski continues to work on updating and improving devicetree files. Johan Hovold worked mostly with a set of Qualcomm PHY drivers, Andy Shevchenko performed cleanups across the driver tree, and Thomas Gleixner worked extensively in the core-kernel and x86 subsystems. 

Usually, when a developer lands at the top of the "lines changed" column, it is the result of adding more machine-generated amdgpu header files. This time, though, Ian Rogers got there by working with the `perf` tool and, in particular, updating a number of [Intel PMU event definition files](<https://elixir.bootlin.com/linux/v6.2-rc8/source/tools/perf/pmu-events/arch/x86>). Ping-Ke Shih contributed a number of improvements to the rtw89 wireless network adapter driver. Ben Skeggs worked, as always, on the Nouveau graphics driver, Steen Hegelund worked on the sparx5 network driver, and Arnd Bergmann deleted a number of unmaintained drivers. 

The top testers and reviewers this time around were: 

> Test and review credits in 6.2  
> ---  
> | Tested-by  
> ---  
> Philipp Hortmann | 154| 11.2%  
> Daniel Wheeler | 94| 6.8%  
> Mark Broadworth | 51| 3.7%  
> Gurucharan G | 39| 2.8%  
> Yang Lixiao | 33| 2.4%  
> Matthew Rosato | 27| 2.0%  
> Vincent Donnefort | 26| 1.9%  
> Yi Liu | 25| 1.8%  
> Nicolin Chen | 24| 1.7%  
> Steev Klimaszewski | 24| 1.7%  
> Yu He | 22| 1.6%  
> Daniel Golle | 20| 1.5%  
> Naveen N. Rao | 16| 1.2%  
> Arnaldo Carvalho de Melo | 16| 1.2%  
> Guenter Roeck | 15| 1.1%  
> | Reviewed-by  
> ---  
> Andy Shevchenko | 271| 3.1%  
> Krzysztof Kozlowski | 249| 2.8%  
> Rob Herring | 231| 2.6%  
> Konrad Dybcio | 196| 2.2%  
> Dmitry Baryshkov | 174| 2.0%  
> AngeloGioacchino Del Regno | 127| 1.4%  
> Lyude Paul | 125| 1.4%  
> Kevin Tian | 118| 1.3%  
> David Sterba | 114| 1.3%  
> Ranjani Sridharan | 108| 1.2%  
> Jason Gunthorpe | 98| 1.1%  
> Hans de Goede | 97| 1.1%  
> Jani Nikula | 93| 1.1%  
> Chaitanya Kulkarni | 89| 1.0%  
> Peter Ujfalusi | 82| 0.9%  
  
Philipp Hortmann's testing was almost entirely limited to Realtek wireless driver changes. Daniel Wheeler continues to test graphics patches from within AMD; Mark Broadworth did the same. In the reviews column, Andy Shevchenko reviewed patches across the driver tree at a rate of nearly four per day. Krzysztof Kozlowski, Rob Herring, and Konrad Dybcio all reviewed devicetree patches at almost the same rate. 

All told, 1,161 commits in 6.2 (7.4%) had Tested-by tags, while 6,735 commits (43.4%) had Reviewed-by tags. A quick look shows that the use of Reviewed-by tags has been steadily growing over the years: 

> Commits with Reviewed-by tags in each release  
> ---  
> Release| Total| Tagged| %|    
> v4.0 | 10,346 | 1,712 | 16.5% |   
> v4.1 | 11,916 | 1,666 | 14.0% |   
> v4.2 | 13,694 | 2,316 | 16.9% |   
> v4.3 | 12,274 | 2,174 | 17.7% |   
> v4.4 | 13,071 | 2,434 | 18.6% |   
> v4.5 | 12,080 | 2,355 | 19.5% |   
> v4.6 | 13,517 | 2,785 | 20.6% |   
> v4.7 | 12,283 | 2,749 | 22.4% |   
> v4.8 | 13,382 | 2,965 | 22.2% |   
> v4.9 | 16,214 | 4,073 | 25.1% |   
> v4.10 | 13,029 | 2,972 | 22.8% |   
> v4.11 | 12,724 | 3,126 | 24.6% |   
> v4.12 | 14,570 | 3,977 | 27.3% |   
> v4.13 | 13,006 | 3,272 | 25.2% |   
> v4.14 | 13,452 | 3,144 | 23.4% |   
> v4.15 | 14,866 | 4,970 | 33.4% |   
> v4.16 | 13,630 | 3,902 | 28.6% |   
> v4.17 | 13,541 | 3,978 | 29.4% |   
> v4.18 | 13,283 | 4,040 | 30.4% |   
> v4.19 | 14,043 | 4,171 | 29.7% |   
> v4.20 | 13,884 | 4,201 | 30.3% |   
> v5.0 | 12,808 | 4,045 | 31.6% |   
> v5.1 | 12,749 | 3,843 | 30.1% |   
> v5.2 | 14,309 | 4,974 | 34.8% |   
> v5.3 | 14,605 | 4,860 | 33.3% |   
> v5.4 | 14,619 | 5,184 | 35.5% |   
> v5.5 | 14,350 | 4,939 | 34.4% |   
> v5.6 | 12,665 | 4,184 | 33.0% |   
> v5.7 | 13,901 | 4,797 | 34.5% |   
> v5.8 | 16,306 | 5,477 | 33.6% |   
> v5.9 | 14,858 | 5,251 | 35.3% |   
> v5.10 | 16,175 | 5,352 | 33.1% |   
> v5.11 | 14,340 | 5,038 | 35.1% |   
> v5.12 | 13,015 | 4,690 | 36.0% |   
> v5.13 | 16,030 | 5,245 | 32.7% |   
> v5.14 | 14,735 | 5,228 | 35.5% |   
> v5.15 | 12,377 | 4,361 | 35.2% |   
> v5.16 | 14,190 | 5,182 | 36.5% |   
> v5.17 | 13,038 | 4,846 | 37.2% |   
> v5.18 | 14,954 | 6,017 | 40.2% |   
> v5.19 | 15,134 | 6,361 | 42.0% |   
> v6.0 | 15,402 | 6,044 | 39.2% |   
> v6.1 | 13,942 | 5,997 | 43.0% |   
> v6.2 | 15,440 | 6,735 | 43.4% |   
  
Less than half of the non-merge commits in 6.2 contain Reviewed-by tags, but the percentage of such tags is still nearly triple what it was for the 4.0 release, almost eight years ago. 

A total of 235 employers supported work on 6.2, a fairly normal sort of number. The most active employers this time were: 

> Most active 6.2 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1658| 10.7%  
> (Unknown)| 1125| 7.2%  
> Google| 1119| 7.2%  
> Linaro| 1118| 7.2%  
> Red Hat| 1026| 6.6%  
> Huawei Technologies| 858| 5.5%  
> Pengutronix| 641| 4.1%  
> AMD| 616| 4.0%  
> (None)| 577| 3.7%  
> SUSE| 457| 2.9%  
> NVIDIA| 443| 2.9%  
> Meta| 429| 2.8%  
> (Consultant)| 323| 2.1%  
> IBM| 249| 1.6%  
> Arm| 240| 1.5%  
> NXP Semiconductors| 237| 1.5%  
> Linutronix| 221| 1.4%  
> Renesas Electronics| 210| 1.4%  
> Microchip Technology Inc.| 172| 1.1%  
> Oracle| 166| 1.1%  
> | By lines changed  
> ---  
> Google| 182057| 20.7%  
> Linaro| 65578| 7.4%  
> Red Hat| 63332| 7.2%  
> Intel| 60882| 6.9%  
> (Unknown)| 41141| 4.7%  
> Realtek| 38961| 4.4%  
> Microchip Technology Inc.| 37217| 4.2%  
> NVIDIA| 33702| 3.8%  
> Meta| 32152| 3.6%  
> AMD| 28888| 3.3%  
> MediaTek| 22373| 2.5%  
> (None)| 22363| 2.5%  
> SUSE| 20743| 2.4%  
> Collabora| 13197| 1.5%  
> Renesas Electronics| 11889| 1.3%  
> Huawei Technologies| 11295| 1.3%  
> IBM| 10225| 1.2%  
> (Consultant)| 8988| 1.0%  
> Analog Devices| 8864| 1.0%  
> NXP Semiconductors| 8829| 1.0%  
  
IBM, once one of the biggest contributors to Linux, continues to slide slowly down in this ranking (Red Hat, which is owned by IBM but said to be run independently, has also declined, but less so). But, in general, this table looks as it always does, without a lot of surprises. 

That reflects the state of the kernel development process as a whole; it continues to produce kernel releases on a predictable schedule without many surprises. As of this writing, there are over 12,000 changesets waiting in linux-next for the 6.3 development cycle, so it looks like the pace will not be slowing down much in the near future. However 6.3 turns out, you'll find the results summarized here shortly thereafter.
