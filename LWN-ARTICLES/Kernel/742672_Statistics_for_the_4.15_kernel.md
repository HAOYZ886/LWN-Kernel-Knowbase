---
title: Statistics for the 4.15 kernel
url: https://lwn.net/Articles/742672/
date: "January 3, 2018"
category: "Releases-4.15"
author: "By Jonathan Corbet January 3, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 3, 2018

The 4.15 kernel is likely to require a relatively long development cycle as a result of the post-rc5 merge of the [kernel page-table isolation](<https://lwn.net/Articles/741878/>) patches. That said, it should be in something close to its final form, modulo some inevitable bug fixes. The development statistics for this kernel release look fairly normal, but they do reveal an unexpectedly busy cycle overall. 

This development cycle was supposed to be relatively calm after the anticipated rush to get work into the 4.14 long-term-support release. But, while 4.14 ended up with 13,452 non-merge changesets at release, 4.15-rc6 already has 14,226, making it one of the busiest releases in the kernel project's history. Only 4.9 (16,214 changesets) and 4.12 (14,570) brought in more work, and 4.15 may exceed 4.12 by the time it is finished. So far, 1,707 developers have contributed to this kernel; they added 725,000 lines of code while removing 407,000, for a net growth of 318,000 lines of code. 

The most active developers this time around were: 

> Most active 4.15 developers  
> ---  
> | By changesets  
> ---  
> Kees Cook| 349| 2.5%  
> Colin Ian King| 237| 1.7%  
> Harry Wentland| 170| 1.2%  
> Ben Skeggs| 156| 1.1%  
> Gustavo A. R. Silva| 138| 1.0%  
> Christoph Hellwig| 137| 1.0%  
> Geert Uytterhoeven| 136| 1.0%  
> Arnd Bergmann| 134| 0.9%  
> Chris Wilson| 129| 0.9%  
> Dmytro Laktyushkin| 125| 0.9%  
> Allen Pais| 112| 0.8%  
> Masahiro Yamada| 108| 0.8%  
> Thomas Gleixner| 105| 0.7%  
> Dave Airlie| 103| 0.7%  
> Eric Dumazet| 99| 0.7%  
> Ville Syrjälä| 97| 0.7%  
> Arvind Yadav| 95| 0.7%  
> Jakub Kicinski| 94| 0.7%  
> Markus Elfring| 92| 0.6%  
> Mauro Carvalho Chehab| 89| 0.6%  
> | By changed lines  
> ---  
> Harry Wentland| 152262| 16.8%  
> Dave Airlie| 47651| 5.2%  
> Takashi Iwai| 41943| 4.6%  
> Dmytro Laktyushkin| 28306| 3.1%  
> Rex Zhu| 24008| 2.6%  
> Andy Shevchenko| 18204| 2.0%  
> Paul E. McKenney| 14629| 1.6%  
> Ben Skeggs| 12684| 1.4%  
> Palmer Dabbelt| 10433| 1.1%  
> David Howells| 10210| 1.1%  
> Darrick J. Wong| 8792| 1.0%  
> Yue Hin Lau| 8483| 0.9%  
> Greg Kroah-Hartman| 8298| 0.9%  
> Kees Cook| 7091| 0.8%  
> Christoph Hellwig| 7076| 0.8%  
> Linus Walleij| 6757| 0.7%  
> Jakub Kicinski| 6402| 0.7%  
> Wei Hu| 5967| 0.7%  
> Mauro Carvalho Chehab| 5692| 0.6%  
> Alex Deucher| 5406| 0.6%  
  
Kees Cook was this cycle's most prolific contributor of changesets; he did security-related work throughout the kernel, but the bulk of the patches implemented the internal [kernel-timer API change](<https://lwn.net/Articles/735887/>). Colin Ian King contributed cleanup patches all over the kernel, Harry Wentland added another massive pile of AMD graphics driver code, Ben Skeggs worked on the Nouveau driver as usual, and Gustavo Silva focused on marking fall-through cases in switch statements (as in [this patch](<https://git.kernel.org/linus/fd53dde83978ba5f7db3183ce56b3a1c39f448b0>)). 

In the lines-changed column, Wentland's AMD graphics driver additions topped the list. Dave Airlie brought [the AMD display core code](<https://lwn.net/Articles/708891/>) into the graphics subsystem, but also did a bunch of cleanup work resulting in the removal of over 21,000 lines of code. Takashi Iwai worked all over the audio subsystem; in particular, he removed the ancient Open Sound System code, shrinking the kernel by over 40,000 lines. Dmytro Laktyushkin and Rex Zhu also added more AMD graphics code. The AMD graphics drivers thus dominated the changes in this cycle in terms of lines of code, as has been the case for a number of recent development cycles. 

It is worth noting once again that staging-tree work hardly figures in these numbers at all; the days when staging was the biggest driver of kernel changes appear to be done. The page-table isolation work also doesn't show up much here either, showing that important changes often come in relatively small packages. 

The work in 4.15 was supported by 231 companies (that we can identify), more than worked on 4.14 but still a relatively small number by recent standards; 4.10 remains the record holder with 271 companies participating. The most active companies this time around were: 

> Most active 4.15 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1609| 11.3%  
> AMD| 1526| 10.7%  
> Red Hat| 955| 6.7%  
> (None)| 813| 5.7%  
> Google| 739| 5.2%  
> (Unknown)| 703| 4.9%  
> Linaro| 489| 3.4%  
> IBM| 450| 3.2%  
> Oracle| 390| 2.7%  
> Renesas Electronics| 343| 2.4%  
> Mellanox| 340| 2.4%  
> Linux Foundation| 307| 2.2%  
> ARM| 306| 2.2%  
> SUSE| 294| 2.1%  
> Broadcom| 260| 1.8%  
> Huawei Technologies| 257| 1.8%  
> Canonical| 254| 1.8%  
> (Consultant)| 251| 1.8%  
> Samsung| 221| 1.6%  
> Netronome Systems| 157| 1.1%  
> | By lines changed  
> ---  
> AMD| 266230| 29.3%  
> Red Hat| 97177| 10.7%  
> Intel| 82791| 9.1%  
> SUSE| 46479| 5.1%  
> (Unknown)| 33739| 3.7%  
> IBM| 33105| 3.6%  
> (None)| 24842| 2.7%  
> Linaro| 23291| 2.6%  
> Google| 17760| 2.0%  
> Broadcom| 15482| 1.7%  
> Mellanox| 14923| 1.6%  
> Samsung| 13841| 1.5%  
> Oracle| 13755| 1.5%  
> Huawei Technologies| 13655| 1.5%  
> ARM| 13118| 1.4%  
> Renesas Electronics| 10762| 1.2%  
> Netronome Systems| 10366| 1.1%  
> Linux Foundation| 9855| 1.1%  
> ST Microelectronics| 8803| 1.0%  
> Chelsio| 8695| 1.0%  
  
The AMD graphics work shows clearly in these numbers; otherwise, the results are typical for recent development cycles. 

The Signed-off-by tags attached to patches give clues as to who took responsibility for their development. In particular, if one looks at the signoffs attached by developers other than the author of the patch, the result is a picture of who accepted the patches for merging into the mainline — the most active maintainers, in other words. For the 4.15 kernel, the results look like this: 

> Non-author signoffs in 4.15  
> ---  
> | By developer  
> ---  
> David S. Miller| 1942| 14.1%  
> Alex Deucher| 1551| 11.3%  
> Greg Kroah-Hartman| 749| 5.5%  
> Ingo Molnar| 397| 2.9%  
> Mark Brown| 329| 2.4%  
> Doug Ledford| 300| 2.2%  
> Mauro Carvalho Chehab| 287| 2.1%  
> Andrew Morton| 271| 2.0%  
> Jens Axboe| 240| 1.7%  
> Martin K. Petersen| 226| 1.6%  
> Thomas Gleixner| 218| 1.6%  
> Simon Horman| 177| 1.3%  
> Herbert Xu| 174| 1.3%  
> Jeff Kirsher| 156| 1.1%  
> Kalle Valo| 152| 1.1%  
> Michael Ellerman| 151| 1.1%  
> Jiri Pirko| 126| 0.9%  
> David Sterba| 114| 0.8%  
> Martin Schwidefsky| 113| 0.8%  
> Linus Walleij| 110| 0.8%  
> | By company  
> ---  
> Red Hat| 3334| 24.3%  
> AMD| 1681| 12.2%  
> Intel| 1088| 7.9%  
> Linaro| 904| 6.6%  
> Linux Foundation| 769| 5.6%  
> Google| 479| 3.5%  
> Samsung| 440| 3.2%  
> Oracle| 395| 2.9%  
> IBM| 372| 2.7%  
> Facebook| 334| 2.4%  
> Huawei Technologies| 328| 2.4%  
> (None)| 320| 2.3%  
> Mellanox| 283| 2.1%  
> SUSE| 270| 2.0%  
> Renesas Electronics| 219| 1.6%  
> Free Electrons| 218| 1.6%  
> Linutronix| 218| 1.6%  
> Code Aurora Forum| 214| 1.6%  
> (Consultant)| 181| 1.3%  
> ARM| 176| 1.3%  
  
Kernel subsystem maintainers have long been concentrated in a relatively small set of companies. That situation is slowly changing, but it's still true that, in 4.15, half of the changes merged were accepted by developers working for just four companies. 

Finally, the most active bug reporters and patch testers, according to the Reported-by and Tested-by tags attached to patches, were: 

> Bug reporters and testers in 4.15  
> ---  
> | Reported-by credits  
> ---  
> kernel test robot| 36| 5.5%  
> Dan Carpenter| 25| 3.8%  
> syzbot| 25| 3.8%  
> Dmitry Vyukov| 12| 1.8%  
> Andrey Konovalov| 11| 1.7%  
> Geert Uytterhoeven| 9| 1.4%  
> Arnd Bergmann| 7| 1.1%  
> Michael Ellerman| 7| 1.1%  
> Randy Dunlap| 7| 1.1%  
> Brian Foster| 7| 1.1%  
> Stephen Rothwell| 7| 1.1%  
> Jianlin Shi| 7| 1.1%  
> Jakub Kicinski| 6| 0.9%  
> | Tested-by credits  
> ---  
> Andrew Bowers| 114| 12.7%  
> Juergen Gross| 52| 5.8%  
> Yu Chen| 51| 5.7%  
> Krishneil Singh| 22| 2.4%  
> Borislav Petkov| 20| 2.2%  
> Oleksandr Natalenko| 16| 1.8%  
> Arnaldo Carvalho de Melo| 15| 1.7%  
> Aaron Brown| 13| 1.4%  
> Sean Wang| 12| 1.3%  
> Chris Brandt| 12| 1.3%  
> Xin Long| 11| 1.2%  
> Geert Uytterhoeven| 9| 1.0%  
> Lee Tibbert| 9| 1.0%  
  
A relatively new entry here is "syzbot", which is an operation run by Dmitry Vyukov at Google. Syzbot runs the [syzkaller](<https://lwn.net/Articles/677764/>) fuzz tester in an automated mode and reports the (numerous) crashes that result. As can be seen in the tags, those reports are leading to a steady stream of bug fixes, which can only be a good thing. 

The story told by that final table is incomplete, though, in that most bug reporting and (especially) most testing goes untracked. The kernel community counts on many people beyond those who directly contribute code; it will never be possible to credit them all. As a whole, this community remains large, active, and growing, and the first kernel to be released in 2018 will reflect that.
