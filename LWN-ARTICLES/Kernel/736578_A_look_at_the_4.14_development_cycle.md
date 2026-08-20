---
title: A look at the 4.14 development cycle
url: https://lwn.net/Articles/736578/
date: "October 20, 2017"
category: "Releases-4.14"
author: "By Jonathan Corbet October 20, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 20, 2017

The 4.14 kernel, due in the first half of November, is moving into the relatively slow part of the development cycle as of this writing. The time is thus ripe for a look at the changes that went into this kernel cycle and how they got there. While 4.14 is a fairly typical kernel development cycle, there are a couple of aspects that stand out this time around. 

As of the 4.14-rc5 prepatch, 12,757 non-merge changesets had found their way into the mainline; that makes 4.14 slightly busier than its predecessor, but it remains a fairly normal development cycle overall. If, [as some have worried](<https://lwn.net/Articles/735902/>), developers have pushed unready code into 4.14 so that it would be present in a long-term-support release, it doesn't show in the overall patch volume. 

1,649 developers have contributed code in this development cycle, a number that will almost certainly increase slightly by the time the final 4.14 release is made. Again, that is up slightly from 4.13. Of those developers, 240 made their first contribution to the kernel in 4.14. The numbers are fairly normal, but a look at the most active developers this time around shows a couple of unusual aspects. 

> Most active 4.14 developers  
> ---  
> | By changesets  
> ---  
> Arvind Yadav| 544| 4.3%  
> Bhumika Goyal| 195| 1.5%  
> Gustavo A. R. Silva| 156| 1.2%  
> Colin Ian King| 151| 1.2%  
> Julia Lawall| 141| 1.1%  
> Arnd Bergmann| 127| 1.0%  
> Mauro Carvalho Chehab| 117| 0.9%  
> Bart Van Assche| 113| 0.9%  
> Arnaldo Carvalho de Melo| 106| 0.8%  
> Paul Burton| 100| 0.8%  
> Chris Wilson| 96| 0.8%  
> Markus Elfring| 94| 0.7%  
> Thomas Gleixner| 87| 0.7%  
> Dan Carpenter| 87| 0.7%  
> Laurent Pinchart| 87| 0.7%  
> Daniel Vetter| 83| 0.7%  
> Xin Long| 82| 0.6%  
> Christoph Hellwig| 79| 0.6%  
> Geert Uytterhoeven| 77| 0.6%  
> Michael Ellerman| 76| 0.6%  
> | By changed lines  
> ---  
> Greg Kroah-Hartman| 129421| 14.9%  
> Ping-Ke Shih| 122912| 14.1%  
> Lionel Landwerlin| 30289| 3.5%  
> Mauro Carvalho Chehab| 22461| 2.6%  
> Daniel Scheller| 17708| 2.0%  
> Nick Terrell| 14223| 1.6%  
> Aviad Krawczyk| 12831| 1.5%  
> Salil Mehta| 12051| 1.4%  
> Juergen Gross| 11036| 1.3%  
> Todor Tomov| 9286| 1.1%  
> Sukadev Bhattiprolu| 9248| 1.1%  
> Hannes Reinecke| 9003| 1.0%  
> Arnaldo Carvalho de Melo| 7790| 0.9%  
> Andi Kleen| 6826| 0.8%  
> Johannes Berg| 6631| 0.8%  
> Masahiro Yamada| 6429| 0.7%  
> Russell King| 4573| 0.5%  
> John Fastabend| 4412| 0.5%  
> Jérôme Glisse| 4128| 0.5%  
> Vikas Shivappa| 4091| 0.5%  
  
Arvind Yadav contributed 544 changesets mostly focused on making device-ID lists in the kernel constant. Unlike much "constification" work, these changes probably do not have much security significance, but they do tend to make the kernel text size a little smaller. Bhumika Goyal got her start as an Outreachy intern making structures full of function pointers `const` — a job that does improve security; she is continuing that work post-Outreachy with support from the Core Infrastructure Initiative (CII). Gustavo A. R. Silva _also_ did constification work, along with contributing a number of other fixes, with CII support. Colin Ian King did, wait for it, constification work along with various other fixes and Julia Lawall also did constification work. 

In other words, the top five contributors contributed nearly 1,200 changes mostly cleaning up declarations in the kernel. This work may not draw the same sort of attention as the addition of hardening mechanisms, but it is an important part of hardening the kernel overall. 

Greg Kroah-Hartman got to the top of the "changed lines" column mostly by virtue of [a single patch](<https://git.kernel.org/linus/5620a0d1aacd554ebebcff373e31107bb1ef7769>) deleting the remaining old firmware files from the kernel tree. Firmware has long been maintained externally, so this cleanup was overdue. Ping-Ke Shih added yet another Realtek wireless driver to the staging tree, Lionel Landwerlin reworked parts of the Intel i915 driver in ways that allowed the removal of a lot of code, Mauro Carvalho Chehab contributed changes all over the media and documentation subsystems as usual, and Daniel Scheller added a large new media driver. 

The 120,000-line Realtek wireless driver merits a bit more attention; it was the subject of [some complaining](<https://lwn.net/Articles/736670/>) about how Realtek gets its drivers into the kernel. But something of note has happened here. Numerous Realtek drivers have been merged via the staging tree, often moving on to the mainline kernel; that work has generally been done by Larry Finger on his own time. But, as he [explained](<https://lwn.net/Articles/736669/>), he is reaching an age where he lacks the energy for this work and intends to stop soon. Thus it is encouraging that Ping-Ke Shih, the submitter of the Realtek driver patch this time around, is an actual Realtek employee. It would appear that the company has finally decided to put some resources into Linux support and, hopefully, the situation with its wireless drivers will improve over time. Meanwhile, Realtek should send Larry a nice retirement present — he has certainly earned it. 

Work on 4.14 was supported by 213 companies that could be identified — a typical number that is, once again, a little higher than the 203 seen in the 4.13 cycle. The most active employers this time around were: 

> Most active 4.14 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1328| 10.4%  
> (None)| 813| 6.4%  
> Red Hat| 754| 5.9%  
> (Unknown)| 575| 4.5%  
> IBM| 566| 4.4%  
> Motorola| 544| 4.3%  
> Linaro| 500| 3.9%  
> Google| 453| 3.6%  
> Mellanox| 425| 3.3%  
> SUSE| 404| 3.2%  
> Linux Foundation| 391| 3.1%  
> AMD| 348| 2.7%  
> Renesas Electronics| 319| 2.5%  
> Samsung| 262| 2.1%  
> Rockchip| 257| 2.0%  
> Oracle| 221| 1.7%  
> ARM| 218| 1.7%  
> (Consultant)| 199| 1.6%  
> Canonical| 185| 1.5%  
> Broadcom| 182| 1.4%  
> | By lines changed  
> ---  
> Linux Foundation| 131369| 15.1%  
> Realtek| 124976| 14.4%  
> Intel| 101671| 11.7%  
> (None)| 47222| 5.4%  
> Red Hat| 31888| 3.7%  
> SUSE| 29408| 3.4%  
> Huawei Technologies| 28807| 3.3%  
> IBM| 28363| 3.3%  
> Linaro| 25614| 2.9%  
> Samsung| 24940| 2.9%  
> (Unknown)| 22749| 2.6%  
> Mellanox| 18379| 2.1%  
> Facebook| 18345| 2.1%  
> Google| 17257| 2.0%  
> AMD| 14621| 1.7%  
> (Consultant)| 12162| 1.4%  
> Renesas Electronics| 12004| 1.4%  
> ST Microelectronics| 9923| 1.1%  
> Rockchip| 9091| 1.0%  
> ARM| 8438| 1.0%  
  
These results are fairly typical for recent kernels; the biggest surprise, perhaps, is the appearance of Realtek as was discussed above. In general, the kernel project continues to move forward powered by a great deal of corporate support. 

While the kernel project clearly depends on developers to get the code written, it also depends on those who test changes and report bugs. Some of the time, at least, that contribution is noted through the addition of tags in the patches. Looking at the `Reported-by` (619 through 4.14-rc5) and `Tested-by` tags (605) yields the following results: 

> Testing and bug reporting in 4.14  
> ---  
> | Reported-by tags  
> ---  
> Fengguang Wu| 36| 5.8%  
> Dan Carpenter| 27| 4.4%  
> Andrey Konovalov| 17| 2.7%  
> Peter Zijlstra| 14| 2.3%  
> Eric Biggers| 12| 1.9%  
> Arnd Bergmann| 11| 1.8%  
> Michael Ellerman| 11| 1.8%  
> Stephen Rothwell| 9| 1.5%  
> kbuild test robot| 7| 1.1%  
> Eric Dumazet| 6| 1.0%  
> Dmitry Vyukov| 6| 1.0%  
> Jianlin Shi| 6| 1.0%  
> Christoph Hellwig| 5| 0.8%  
> Geert Uytterhoeven| 5| 0.8%  
> Ingo Molnar| 5| 0.8%  
> Tetsuo Handa| 5| 0.8%  
> kernel test robot| 5| 0.8%  
> Mathias Kresin| 5| 0.8%  
> | Tested-by tags  
> ---  
> Andrew Bowers| 51| 8.4%  
> Richard Scobie| 21| 3.5%  
> Arnaldo Carvalho de Melo| 19| 3.1%  
> Laurent Pinchart| 16| 2.6%  
> Thierry Reding| 16| 2.6%  
> Andrey Konovalov| 15| 2.5%  
> Laura Abbott| 14| 2.3%  
> Eric Biggers| 12| 2.0%  
> Jasmin Jessich| 12| 2.0%  
> Dietmar Spingler| 12| 2.0%  
> Manfred Knick| 12| 2.0%  
> Aaron Brown| 12| 2.0%  
> Marcin Wojtas| 12| 2.0%  
> Pavel Machek| 8| 1.3%  
> Philippe Cornu| 7| 1.2%  
> Stan Johnson| 7| 1.2%  
> Heiko Stuebner| 6| 1.0%  
> Ondrej Zary| 6| 1.0%  
  
In truth, the report credits for Fengguang Wu and the two "test robot" entries probably belong together, but they were credited separately in the patches. 

These numbers, of course, greatly understate the amount of testing and reporting that is happening in the kernel community. The requisite tags do not always get added to patches as they should and, in the case of testing, many testers do not make themselves known in the first place if things work for them. That said, the tags that do exist record real work done, and the kernel is better for it. 

Overall, the kernel development process would appear to continue to run relatively smoothly, despite the occasional hiccup. There are few projects that would be able to integrate changes at this rate and produce a result that will be the base for countless deployed systems in the coming years.
