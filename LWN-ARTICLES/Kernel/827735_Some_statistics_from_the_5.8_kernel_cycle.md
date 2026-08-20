---
title: Some statistics from the 5.8 kernel cycle
url: https://lwn.net/Articles/827735/
date: "August 3, 2020"
category: "Releases-5.8"
author: "By Jonathan Corbet August 3, 2020"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 3, 2020

Linus Torvalds [released the 5.8 kernel](<https://lwn.net/ml/linux-kernel/CAHk-=wj+mDPbj8hXspXRAksh+1TmPjubc9RNEbu8EVpYyypX=w@mail.gmail.com/>) on August 2, concluding another nine-week development cycle. By the time the work was done, 16,306 non-merge changesets had been pulled into the mainline repository for this release. That happens to be a record, beating the previous record holder (4.9, released in December 2016) by 92 changesets. It was, in other words, a busy development cycle. It's time for our traditional look into where that work came from to see what might be learned. 

A total of 1,991 developers contributed to 5.8, which is another record; 304 of those developers appeared for the first time in this cycle. The community added over 924,000 lines of code and removed around 371,000 for a net growth of over 553,000 lines of code. The most active developers for 5.8 were: 

> Most active 5.8 developers  
> ---  
> | By changesets  
> ---  
> Mauro Carvalho Chehab| 549| 3.4%  
> Christoph Hellwig| 354| 2.2%  
> Andy Shevchenko| 223| 1.4%  
> Jason Yan| 205| 1.3%  
> Chris Wilson| 199| 1.2%  
> Jérôme Pouiller| 175| 1.1%  
> Thomas Gleixner| 156| 1.0%  
> Gustavo A. R. Silva| 136| 0.8%  
> Masahiro Yamada| 133| 0.8%  
> Miquel Raynal| 125| 0.8%  
> Leon Romanovsky| 114| 0.7%  
> Sean Christopherson| 109| 0.7%  
> Geert Uytterhoeven| 101| 0.6%  
> Colin Ian King| 101| 0.6%  
> Daniel Vetter| 99| 0.6%  
> Al Viro| 98| 0.6%  
> Peter Zijlstra| 95| 0.6%  
> Christophe Leroy| 93| 0.6%  
> Lorenzo Bianconi| 89| 0.5%  
> Serge Semin| 87| 0.5%  
> | By changed lines  
> ---  
> Mauro Carvalho Chehab| 272614| 25.8%  
> Oded Gabbay| 80603| 7.6%  
> Yan-Hsuan Chuang| 15798| 1.5%  
> Arnd Bergmann| 13082| 1.2%  
> Jack Wang| 12895| 1.2%  
> Thomas Bogendoerfer| 11161| 1.1%  
> Christoph Hellwig| 10940| 1.0%  
> Omer Shpigelman| 10861| 1.0%  
> Ryder Lee| 10076| 1.0%  
> Chris Wilson| 8682| 0.8%  
> David Howells| 8130| 0.8%  
> Serge Semin| 7520| 0.7%  
> Andrii Nakryiko| 6189| 0.6%  
> Thomas Gleixner| 5695| 0.5%  
> Marco Elver| 5619| 0.5%  
> Peter Zijlstra| 5533| 0.5%  
> Boris Brezillon| 5451| 0.5%  
> Leon Romanovsky| 5399| 0.5%  
> Ping-Ke Shih| 5173| 0.5%  
> Bryan O'Donoghue| 4953| 0.5%  
  
In this cycle, Mauro Carvalho Chehab managed to get to the top of both the by-changesets and by-lines columns. Much of his work was focused on documentation, converting more files to RST and reworking the video4linux2 user-space manual, but he also put a lot of work into resurrecting the atomisp camera driver, which had been removed from the staging tree. Christoph Hellwig has done significant work throughout the kernel's memory-management, filesystem, and block subsystems. Andy Shevchenko improved a number of different drivers, Jason Yan performed code cleanups across the kernel, and Chris Wilson, as usual, did a lot of work on the i915 graphics driver. 

In the lines-changed column, Oded Gabbay added [a massive set of machine-generated register definitions](<https://git.kernel.org/linus/2aad2bf81c33>) for the Habana Gaudi processor. Yan-Hsuang added [a set of machine-generated data](<https://elixir.bootlin.com/linux/latest/source/drivers/net/wireless/realtek/rtw88/rtw8822c_table.c>) for the Realtek rtw88 wireless driver that looks rather more like binary code than source. Arnd Bergmann did cleanup work all over as usual; part of that work was deleting support for the never-realized sh5 subarchitecture. Jack Wang contributed the rndb (a network block device using RDMA) driver. 

While the number of developers contributing to the kernel set a new record, the number of companies supporting them remains about flat at 213. The companies supporting the most work in the 5.8 cycle were: 

> Most active 5.8 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1939| 11.9%  
> Huawei Technologies| 1399| 8.6%  
> (Unknown)| 1231| 7.5%  
> Red Hat| 1079| 6.6%  
> (None)| 1016| 6.2%  
> Google| 791| 4.9%  
> IBM| 542| 3.3%  
> (Consultant)| 515| 3.2%  
> Linaro| 513| 3.1%  
> AMD| 503| 3.1%  
> SUSE| 463| 2.8%  
> Mellanox| 445| 2.7%  
> NXP Semiconductors| 330| 2.0%  
> Renesas Electronics| 322| 2.0%  
> Oracle| 252| 1.5%  
> Code Aurora Forum| 248| 1.5%  
> Facebook| 247| 1.5%  
> Arm| 239| 1.5%  
> Silicon Labs| 175| 1.1%  
> Linux Foundation| 171| 1.0%  
> | By lines changed  
> ---  
> Huawei Technologies| 293365| 27.8%  
> Habana Labs| 93213| 8.8%  
> Intel| 88288| 8.4%  
> (None)| 47655| 4.5%  
> (Unknown)| 36786| 3.5%  
> Linaro| 36322| 3.4%  
> Red Hat| 34737| 3.3%  
> Google| 34209| 3.2%  
> IBM| 24233| 2.3%  
> Mellanox| 23364| 2.2%  
> Realtek| 22767| 2.2%  
> AMD| 21411| 2.0%  
> NXP Semiconductors| 21328| 2.0%  
> (Consultant)| 15418| 1.5%  
> Facebook| 14874| 1.4%  
> MediaTek| 14751| 1.4%  
> SUSE| 13659| 1.3%  
> 1&1 IONOS Cloud| 13219| 1.3%  
> Code Aurora Forum| 11865| 1.1%  
> Renesas Electronics| 11077| 1.1%  
  
For the most part, this table looks fairly familiar, but the fact that Huawei has moved up to the top of the list may come as a bit of a surprise. Much of this is the result of Chehab's work described above, but Huawei's contribution this time around is rather larger than that. A great deal of effort has gone into freezing Huawei out of the commercial marketplace in significant parts of the world, but the company remains active in the development community with 92 developers contributing to 5.8. For the curious, Huawei's work was mostly focused in these subsystems: 

> Subsystem| Changesets  
> ---|---  
> `Documentation`| 226  
> `drivers/net`| 226  
> `drivers/staging`| 222  
> `fs`| 73  
> `drivers/media`| 62  
> `drivers/scsi`| 62  
> `drivers/gpu`| 49  
> `net`| 49  
> `include`| 38  
> `sound`| 22  
> `security`| 21  
> `kernel`| 18  
  
In summary, 907 of the patches coming from Huawei (65% of the total) applied somewhere in the driver subsystem, but quite a bit of the company's work was spread out over the rest of the kernel as well. 

The kernel depends on people who run tests and report bugs; no kernel developer can hope to test every hardware combination and workload out there. The most active contributors in this area in 5.8 were: 

> Test and report credits in 5.8   
> ---  
> | Tested-by  
> ---  
> Aaron Brown| 97| 9.1%  
> Andrew Bowers| 90| 8.5%  
> Arnaldo Carvalho de Melo| 53| 5.0%  
> Hoan Tran| 21| 2.0%  
> Marek Szyprowski| 19| 1.8%  
> Serge Semin| 16| 1.5%  
> David Heidelberg| 14| 1.3%  
> Peter Geis| 14| 1.3%  
> Jasper Korten| 13| 1.2%  
> Tomasz Maciej Nowak| 12| 1.1%  
> | Reported-by  
> ---  
> Hulk Robot| 243| 19.8%  
> kernel test robot| 178| 14.5%  
> Syzbot| 70| 5.7%  
> Dan Carpenter| 33| 2.7%  
> Stephen Rothwell| 26| 2.1%  
> Randy Dunlap| 20| 1.6%  
> Guenter Roeck| 13| 1.1%  
> Qian Cai| 11| 0.9%  
> Greg Kroah-Hartman| 8| 0.7%  
> Lars-Peter Clausen| 8| 0.7%  
  
Automated testing systems continue to report (by far) the most bugs, but this important work is not limited to such systems. 

Patch review is also important, that's how we keep bugs from needing to be reported in the first place. While not all review results in a Reviewed-by tag, there is still a signal to be seen by looking at those tags: 

> Review credits in 5.8  
> ---  
> Rob Herring| 183| 2.6%  
> Christoph Hellwig| 179| 2.6%  
> Alexandre Chartre| 128| 1.8%  
> Andy Shevchenko| 125| 1.8%  
> Ranjani Sridharan| 121| 1.7%  
> Andrew Lunn| 113| 1.6%  
> Darrick J. Wong| 107| 1.5%  
> Florian Fainelli| 94| 1.4%  
> Jiri Pirko| 88| 1.3%  
> David Sterba| 83| 1.2%  
> Hannes Reinecke| 81| 1.2%  
> Ursula Braun| 79| 1.1%  
> Alex Deucher| 78| 1.1%  
> Stephen Boyd| 78| 1.1%  
> Kees Cook| 78| 1.1%  
  
Of the patches merged for 5.8, 5,470 (34% of the total) carried Reviewed-by tags. The last few kernel releases have consistently had such tags in almost exactly one-third of the changesets merged. 

The end result of all this is that the kernel-development community continues to work at a high rate. If the ongoing pandemic has had any effect at all, it would appear to have made things go even faster. It will be interesting to see if this trend continues into the 5.9 development cycle; tune in at the beginning of October for the answer to that question.
