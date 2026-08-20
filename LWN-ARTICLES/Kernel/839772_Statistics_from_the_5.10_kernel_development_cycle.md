---
title: Statistics from the 5.10 kernel development cycle
url: https://lwn.net/Articles/839772/
date: "December 14, 2020"
category: "Releases-5.10"
author: "By Jonathan Corbet December 14, 2020"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 14, 2020

Linus Torvalds [released](<https://lwn.net/ml/linux-kernel/CAHk-=whCKhxNyKn1Arut8xUDKTwp3fWcCj_jbL5dbzkUmo45gQ@mail.gmail.com/>) the 5.10 kernel on December 13 at the end of a typical nine-week development cycle. At that point, 16,174 non-merge changesets had been pulled into the mainline; that makes 5.10 a larger cycle than 5.9, but it falls just short of the record set by 5.8, which ended with 16,308 changesets. For the most part 5.10 is just another routine kernel release, but there are a couple of interesting things to be seen in the overall statistics. 

A total of 1,971 developers contributed to 5.10 — again, just short of the record set by 5.8. Of those developers, 252 (just under 13%) made their first contribution in 5.10; that is the lowest number seen since 5.6. The most active 5.10 developers were: 

> Most active 5.10 developers  
> ---  
> | By changesets  
> ---  
> Krzysztof Kozlowski| 491| 3.0%  
> Mauro Carvalho Chehab| 378| 2.3%  
> Christoph Hellwig| 265| 1.6%  
> Pierre-Louis Bossart| 116| 0.7%  
> Lee Jones| 116| 0.7%  
> Randy Dunlap| 115| 0.7%  
> Allen Pais| 112| 0.7%  
> Jonathan Cameron| 107| 0.7%  
> Maxime Ripard| 103| 0.6%  
> Dave Airlie| 98| 0.6%  
> Lad Prabhakar| 97| 0.6%  
> Andy Shevchenko| 87| 0.5%  
> Chris Wilson| 85| 0.5%  
> Evan Quan| 84| 0.5%  
> Colin Ian King| 84| 0.5%  
> Andrii Nakryiko| 82| 0.5%  
> Vladimir Oltean| 80| 0.5%  
> Alex Deucher| 79| 0.5%  
> Qinglang Miao| 77| 0.5%  
> Kees Cook| 70| 0.4%  
> | By changed lines  
> ---  
> Sudeep Dutt| 26779| 3.5%  
> Mauro Carvalho Chehab| 22741| 2.9%  
> Corentin Labbe| 19091| 2.5%  
> Fabio Estevam| 16757| 2.2%  
> Christoph Hellwig| 14286| 1.9%  
> Cezary Rojewski| 14106| 1.8%  
> Chandan Uddaraju| 9357| 1.2%  
> Daniel W. S. Almeida| 8012| 1.0%  
> Mike Travis| 7873| 1.0%  
> Andrii Nakryiko| 7455| 1.0%  
> Oded Gabbay| 6989| 0.9%  
> Hans Verkuil| 6839| 0.9%  
> Larry Finger| 6758| 0.9%  
> Vadym Kochan| 6382| 0.8%  
> Krzysztof Kozlowski| 6371| 0.8%  
> Mauro Rossi| 6227| 0.8%  
> Jonathan Marek| 6106| 0.8%  
> Marc Kleine-Budde| 6049| 0.8%  
> Jin Yao| 5811| 0.8%  
> Jiaxin Yu| 5464| 0.7%  
  
The author contributing the most changesets to 5.10 was Krzysztof Kozlowski, who made cleanups and small improvements all over the Arm and driver subsystems — at a rate of almost eight per day, seven days per week. Mauro Carvalho Chehab did a lot of work across the media, documentation, and staging subsystems. Christoph Hellwig's work included significant rewrites across the filesystem and block layers, along with the [`set_fs()` removal](<https://lwn.net/Articles/832121/>). Pierre-Louis Bossart did a lot of work in the sound subsystem, and Lee Jones contributed a large number of warning fixes. 

Sudeep Dutt contributed [a single patch](<https://git.kernel.org/linus/80ade22c06ca>) to 5.10, but that patch removed the drivers for Intel "many integrated core" (MIC) devices, deleting enough code to land at the top of the "lines changed" column. Corentin Labbe resurrected the [Zoran MJPEG capture driver](<https://www.kernel.org/doc/html/v4.10/media/v4l-drivers/zoran.html>) in the staging tree, and Fabio Estevam removed a bunch of old Arm board files. 

The busiest testers and bug reporters this time were: 

> Test and report credits in 5.10   
> ---  
> | Tested-by  
> ---  
> Hoegeun Kwon| 86| 7.3%  
> Chanwoo Choi| 79| 6.7%  
> Stefan Wahren| 79| 6.7%  
> Thierry Reding| 33| 2.8%  
> Aaron Brown| 31| 2.6%  
> Arnaldo Carvalho de Melo| 24| 2.0%  
> Nikolay Borisov| 23| 1.9%  
> Nick Desaulniers| 21| 1.8%  
> Yoshihiro Shimoda| 16| 1.4%  
> Jonas Karlman| 16| 1.4%  
> Douglas Gilbert| 16| 1.4%  
> Marek Szyprowski| 15| 1.3%  
> Srinivas Kandagatla| 13| 1.1%  
> Sedat Dilek| 13| 1.1%  
> Daniel Thompson| 12| 1.0%  
> | Reported-by  
> ---  
> kernel test robot| 191| 15.1%  
> Hulk Robot| 190| 15.0%  
> Syzbot| 97| 7.7%  
> Dan Carpenter| 40| 3.2%  
> Stephen Rothwell| 23| 1.8%  
> Randy Dunlap| 20| 1.6%  
> Qian Cai| 17| 1.3%  
> Naresh Kamboju| 14| 1.1%  
> Julien Grall| 8| 0.6%  
> Alexei Starovoitov| 7| 0.6%  
> Rob Herring| 7| 0.6%  
> Marek Szyprowski| 7| 0.6%  
> Colin Ian King| 7| 0.6%  
> Geert Uytterhoeven| 6| 0.5%  
> Lars-Peter Clausen| 6| 0.5%  
  
The top three testers show an interesting pattern: their Tested-by tags all appear together on the same patches, all targeting the vc4 DRM driver. On the report side, we see that nearly 38% of all credited bug reports come from automated testing systems. Note that, since these tags appear in patches, they indicate reports that actually resulted in some sort of fix; that is a lot of bugs that won't be around to affect users later on. 

A total of 228 companies (that we know of) supported work on the 5.10 kernel, an increase relative to recent past releases. The companies supporting the most work were: 

> Most active 5.10 employers  
> ---  
> | By changesets  
> ---  
> Huawei Technologies| 1434| 8.9%  
> Intel| 1297| 8.0%  
> (Unknown)| 1075| 6.6%  
> (None)| 954| 5.9%  
> Red Hat| 915| 5.7%  
> Google| 848| 5.2%  
> AMD| 698| 4.3%  
> Linaro| 670| 4.1%  
> Samsung| 570| 3.5%  
> IBM| 521| 3.2%  
> NXP Semiconductors| 439| 2.7%  
> Facebook| 422| 2.6%  
> Oracle| 414| 2.6%  
> SUSE| 410| 2.5%  
> (Consultant)| 404| 2.5%  
> Code Aurora Forum| 313| 1.9%  
> Arm| 307| 1.9%  
> Renesas Electronics| 283| 1.7%  
> NVIDIA| 262| 1.6%  
> Texas Instruments| 218| 1.3%  
> | By lines changed  
> ---  
> Intel| 96976| 12.6%  
> Huawei Technologies| 41049| 5.3%  
> (Unknown)| 40948| 5.3%  
> Google| 39160| 5.1%  
> NXP Semiconductors| 35898| 4.7%  
> (None)| 30998| 4.0%  
> Red Hat| 30467| 3.9%  
> Code Aurora Forum| 29615| 3.8%  
> Linaro| 29384| 3.8%  
> Facebook| 27479| 3.6%  
> BayLibre| 24159| 3.1%  
> AMD| 23343| 3.0%  
> (Consultant)| 19905| 2.6%  
> IBM| 18312| 2.4%  
> MediaTek| 15893| 2.1%  
> Arm| 13390| 1.7%  
> Texas Instruments| 11814| 1.5%  
> SUSE| 11063| 1.4%  
> Oracle| 10542| 1.4%  
> NVIDIA| 10481| 1.4%  
  
The presence of Huawei at the top of the "by changesets" column may be a bit of a surprise, though something similar [happened in 5.8](<https://lwn.net/Articles/827735/>). As was the case then, Chehab's work obviously helped to drive that number, but it was also the result of 94 other developers working for Huawei who contributed at least one patch to 5.10. Huawei has built up a significant kernel-development operation. Beyond that, these results are mostly as one would expect. 

Another difference in this cycle can be seen by looking at the non-author signoffs in the merged commits. Applying a Signed-off-by tag to a patch that one did not write is usually done by maintainers who are applying patches to be sent upstream; looking at these signoffs thus give an indication of who the gatekeepers to the kernel are. For 5.10, the results look like this: 

> Non-author signoffs in 5.10  
> ---  
> | Developers  
> ---  
> David S. Miller| 1074| 7.0%  
> Greg Kroah-Hartman| 780| 5.1%  
> Mark Brown| 707| 4.6%  
> Alex Deucher| 609| 4.0%  
> Jakub Kicinski| 486| 3.2%  
> Mauro Carvalho Chehab| 485| 3.1%  
> Kalle Valo| 453| 2.9%  
> Andrew Morton| 423| 2.7%  
> Jens Axboe| 329| 2.1%  
> Alexei Starovoitov| 317| 2.1%  
> Hans Verkuil| 315| 2.0%  
> Martin K. Petersen| 289| 1.9%  
> Michael Ellerman| 245| 1.6%  
> Vinod Koul| 239| 1.6%  
> Shawn Guo| 197| 1.3%  
> Paolo Bonzini| 196| 1.3%  
> Borislav Petkov| 187| 1.2%  
> David Sterba| 176| 1.1%  
> Herbert Xu| 175| 1.1%  
> Will Deacon| 172| 1.1%  
> | Employers  
> ---  
> Red Hat| 2198| 14.3%  
> Linaro| 1588| 10.3%  
> Facebook| 1214| 7.9%  
> Intel| 1143| 7.4%  
> Google| 1082| 7.0%  
> Linux Foundation| 863| 5.6%  
> Huawei Technologies| 788| 5.1%  
> SUSE| 710| 4.6%  
> AMD| 642| 4.2%  
> Code Aurora Forum| 508| 3.3%  
> IBM| 418| 2.7%  
> Oracle| 407| 2.6%  
> (None)| 392| 2.5%  
> NVIDIA| 364| 2.4%  
> Cisco| 315| 2.0%  
> Arm| 296| 1.9%  
> Qualcomm| 239| 1.6%  
> (Consultant)| 221| 1.4%  
> Texas Instruments| 187| 1.2%  
> Samsung| 138| 0.9%  
  
Seeing David Miller, the maintainer of the networking subsystem, at the top of this list is traditional, and he occupies this position for 5.10 as well. Miller, unfortunately, abruptly dropped out of the kernel community just after the 5.9 release due to [a health issue](<https://lwn.net/Articles/834026/>), so this table reflects his work done prior to the 5.10 merge window. After that, Jakub Kicinski took over networking maintenance, which explains his appearance on the list (and Facebook's relatively high position). This change will be more strongly felt in the 5.11 release; Kicinski currently has signed off nearly 900 patches in linux-next. 

Happily, Miller is recovering and has started applying networking patches again. Meanwhile, this episode turned into an unplanned test of the community's response when one of its most active maintainers is no longer able to do that work. That response appears to have gone well, with the flow of networking patches into the mainline and linux-next continuing at a strong pace. There appears to have been little disruption in the networking community overall. 

It would be nice to believe that all important kernel subsystems are as well prepared for an abrupt maintainer change, but that is almost certainly not the case. So the kernel community might not pass the next test in such good form. That said, the numbers this time around show that kernel development is still going strong; kernels are released on a predictable schedule, participation across the community is high, and there are still numerous new developers making their debut in each release. 2020 was a difficult year, but the kernel community has gotten through it with, it seems, relatively little trouble.
