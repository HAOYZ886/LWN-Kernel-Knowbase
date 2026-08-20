---
title: Development statistics for the 5.7 kernel
url: https://lwn.net/Articles/821813/
date: "June 2, 2020"
category: "Releases-5.7"
author: "By Jonathan Corbet June 2, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 2, 2020

The 5.7 kernel was [released](<https://lwn.net/Articles/821872/>) on May 31\. By all appearances this was a normal development cycle, unaffected by the troubles in the wider world. Still, there are things to be learned by looking at where the code came from this time around. Read on for LWN's traditional look at who contributed to 5.7, who supported that work, and the paths by which it got into the mainline. 

Work on 5.7 arrived in the form of 13,901 non-merge changesets contributed by 1,878 developers; that makes it rather busier than the 5.6 cycle was. It's notable that 281 of those developers made their first contribution to the kernel for 5.7, the highest number since 5.0; that is a distinct contrast from 5.6, which saw the lowest number of new contributors since 2013\. Perhaps being made to stay at home has inspired more people to put together and send in that first kernel patch. 

The most active developers contributing to 5.7 were: 

> Most active 5.7 developers  
> ---  
> | By changesets  
> ---  
> Gustavo A. R. Silva| 235| 1.7%  
> Chris Wilson| 231| 1.7%  
> Geert Uytterhoeven| 161| 1.2%  
> Christoph Hellwig| 138| 1.0%  
> Sean Christopherson| 137| 1.0%  
> Takashi Iwai| 132| 0.9%  
> Mauro Carvalho Chehab| 129| 0.9%  
> Anson Huang| 109| 0.8%  
> Al Viro| 108| 0.8%  
> Andy Shevchenko| 101| 0.7%  
> Ville Syrjälä| 98| 0.7%  
> Kuninori Morimoto| 96| 0.7%  
> Jani Nikula| 95| 0.7%  
> Thomas Gleixner| 91| 0.7%  
> Colin Ian King| 91| 0.7%  
> Masahiro Yamada| 90| 0.6%  
> Lorenzo Bianconi| 90| 0.6%  
> Jakub Kicinski| 86| 0.6%  
> Ard Biesheuvel| 85| 0.6%  
> Josef Bacik| 83| 0.6%  
> | By changed lines  
> ---  
> Greg Kroah-Hartman| 41035| 6.4%  
> Alex Elder| 14405| 2.3%  
> Chris Packham| 10886| 1.7%  
> Mauro Carvalho Chehab| 10355| 1.6%  
> Chris Wilson| 7931| 1.2%  
> Jani Nikula| 7719| 1.2%  
> Marc Zyngier| 7659| 1.2%  
> Srujana Challa| 7537| 1.2%  
> Namjae Jeon| 7269| 1.1%  
> Manivannan Sadhasivam| 6836| 1.1%  
> Jyri Sarha| 5622| 0.9%  
> Linus Walleij| 5056| 0.8%  
> Christoph Hellwig| 4957| 0.8%  
> Laurent Pinchart| 4781| 0.7%  
> Taniya Das| 4714| 0.7%  
> Paul Blakey| 4367| 0.7%  
> Dmitry Bogdanov| 4328| 0.7%  
> Vladimir Oltean| 4210| 0.7%  
> Jerome Brunet| 3973| 0.6%  
> Maxime Jourdan| 3921| 0.6%  
  
Gustavo A. R. Silva's place at the top of the "by changesets" column is almost entirely a result of his ongoing mission to replace zero-length arrays in structures with flexible arrays throughout the kernel; see [this commit](<https://git.kernel.org/linus/8695e0b1b964>) for a typical example of this work. Chris Wilson worked exclusively on the Intel i915 graphics driver, Geert Uytterhoeven contributed changes throughout various driver subsystems, Christoph Hellwig worked extensively in the XFS, SCSI, and block subsystems (and beyond), and Sean Christopherson continues to do a lot of work with the KVM hypervisor. 

When Greg Kroah-Hartman gets to the top of the "lines changed" column, it usually means he's been busy deleting code, and that is certainly the case this time; he removed the exFAT filesystem (which reappeared later in the filesystem tree) and the wireless USB and UWB drivers from the staging tree. Alex Elder contributed the Qualcomm "IP accelerator" network driver, Chris Packham restored the Octeon USB and Ethernet drivers to the staging tree, and Mauro Carvalho Chehab converted untold numbers of documents to the RST format. 

Work in 5.7 was supported by 215 employers that we were able to identify. The most active of those employers were: 

> Most active 5.7 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1682| 12.1%  
> (Unknown)| 1202| 8.6%  
> Red Hat| 986| 7.1%  
> (None)| 788| 5.7%  
> SUSE| 548| 3.9%  
> Google| 514| 3.7%  
> Huawei Technologies| 508| 3.7%  
> Mellanox| 492| 3.5%  
> AMD| 491| 3.5%  
> Linaro| 412| 3.0%  
> (Consultant)| 386| 2.8%  
> NXP Semiconductors| 374| 2.7%  
> IBM| 371| 2.7%  
> Renesas Electronics| 315| 2.3%  
> Linux Foundation| 310| 2.2%  
> Arm| 278| 2.0%  
> Facebook| 192| 1.4%  
> Code Aurora Forum| 181| 1.3%  
> Oracle| 176| 1.3%  
> Texas Instruments| 175| 1.3%  
> | By lines changed  
> ---  
> Intel| 69584| 10.9%  
> Linux Foundation| 45153| 7.1%  
> Linaro| 44649| 7.0%  
> (Unknown)| 40631| 6.4%  
> Red Hat| 33022| 5.2%  
> (None)| 20662| 3.2%  
> Google| 19940| 3.1%  
> (Consultant)| 19425| 3.0%  
> Mellanox| 19317| 3.0%  
> SUSE| 19127| 3.0%  
> Huawei Technologies| 18301| 2.9%  
> Code Aurora Forum| 17861| 2.8%  
> Marvell| 17833| 2.8%  
> Texas Instruments| 17314| 2.7%  
> IBM| 14973| 2.3%  
> NXP Semiconductors| 14223| 2.2%  
> AMD| 12787| 2.0%  
> BayLibre| 11445| 1.8%  
> Samsung| 11238| 1.8%  
> Allied Telesis| 11029| 1.7%  
  
As usual, there is little change in this table from one development cycle to the next. 

#### How those changes get into the kernel

The days when developers would send their patches directly to Linus Torvalds are long past; almost all work passes through the hands of one or more subsystem maintainers first. All maintainers who manage a patch until (and including) its merging into a subsystem repository will add a Signed-off-by tag to that patch. That helps to identify the chain that got a particular patch into the mainline; it is also useful for generating metrics about who is managing patches in general. A Signed-off-by tag by a developer who is not the author of a patch is almost always indicative of subsystem maintainer activity, so looking at such tags tells us who those maintainers are. 

In 5.7 the busiest maintainers (as measured by non-author Signed-off-by tags) and the employers that supported them were: 

> Non-author signoffs in 5.7  
> ---  
> | Developers  
> ---  
> David S. Miller| 1531| 11.6%  
> Greg Kroah-Hartman| 811| 6.1%  
> Mark Brown| 538| 4.1%  
> Alex Deucher| 429| 3.2%  
> Andrew Morton| 401| 3.0%  
> Martin K. Petersen| 278| 2.1%  
> Jens Axboe| 250| 1.9%  
> Mauro Carvalho Chehab| 236| 1.8%  
> Paolo Bonzini| 235| 1.8%  
> Shawn Guo| 213| 1.6%  
> David Sterba| 196| 1.5%  
> Herbert Xu| 170| 1.3%  
> Michael Ellerman| 169| 1.3%  
> Alexei Starovoitov| 168| 1.3%  
> Saeed Mahameed| 158| 1.2%  
> Vinod Koul| 158| 1.2%  
> Hans Verkuil| 157| 1.2%  
> Ingo Molnar| 146| 1.1%  
> Jason Gunthorpe| 145| 1.1%  
> Thomas Gleixner| 143| 1.1%  
> | Employers  
> ---  
> Red Hat| 2560| 19.3%  
> Linaro| 1377| 10.4%  
> Intel| 986| 7.4%  
> Linux Foundation| 878| 6.6%  
> Google| 787| 5.9%  
> Huawei Technologies| 488| 3.7%  
> Mellanox| 486| 3.7%  
> SUSE| 486| 3.7%  
> Facebook| 465| 3.5%  
> AMD| 463| 3.5%  
> (None)| 440| 3.3%  
> Oracle| 411| 3.1%  
> IBM| 347| 2.6%  
> Texas Instruments| 231| 1.7%  
> Arm| 231| 1.7%  
> Code Aurora Forum| 213| 1.6%  
> (Unknown)| 200| 1.5%  
> Qualcomm| 159| 1.2%  
> (Consultant)| 158| 1.2%  
> Cisco| 158| 1.2%  
  
There may be over 200 companies supporting work on the Linux kernel, but it is still true that half of the patches going into the kernel pass through the hands of gatekeepers working for just five companies. 

Once a patch lands in a subsystem Git repository, the trail of Signed-off-by tags ends. With a bit of digging, though, one can still find the traces that are left when a branch from one repository is merged into another; that allows the generation of a picture showing the paths taken by patches once they are applied. The result is a complicated graph, a small piece of which looks like this: 

> [ ![\[Tree
plot\]](https://static.lwn.net/images/2020/5.7-treeplot-sm.png)](<https://lwn.net/Articles/821973/>)

Click on the image above to see the entire graph in its full, eye-straining glory. 

One picture that emerges quickly from the graph is that it is still fairly flat. Some large subsystems go through a few layers of maintainers, but there are a lot of trees that feed directly into the mainline. Developers may not send their patches straight to Torvalds anymore, but subsystem maintainers still do. 

There is another thing to be seen in this plot. Maintainers are supposed to apply cryptographic signatures to their tags before pushing changes upstream; that is how the recipient knows that the pull request actually came from the person it appears to. Recently, Torvalds [asked a maintainer](<https://lwn.net/ml/linux-kernel/CAHk-=wh_Zw7ug+iMALAKfQkdyVAUWC0UB0bfRRPMOCC7U5uTFQ@mail.gmail.com/>) to start using signed tags, and added that ""I've been encouraging people to do that even on kernel.org, and we've got fairly high coverage these days"". One is naturally led to wonder how high that coverage actually is. 

In the plot, trees that are not using signed tags to push changes upstream are colored red; one sees that there are still quite a few of them. Of the 121 trees that Torvalds pulled from during the 5.7 development cycle, 101 were using signed tags, while 20 were not, so that coverage is just over 83%. That only looks at trees directly pulled by Torvalds, though. If one looks at the whole picture, there are 214 subsystem trees involved, of which 167 are using signed tags — 78% of the total. So coverage may be "fairly high", but it is certainly not universal yet. 

But, then, the kernel development process has always been a work in progress; like the kernel itself, it will never reach a point where it can be deemed to be "perfect". Meanwhile, it continues to integrate changes at a high rate and release the kernels that we all depend on; perfection is not needed to be good enough most of the time.
