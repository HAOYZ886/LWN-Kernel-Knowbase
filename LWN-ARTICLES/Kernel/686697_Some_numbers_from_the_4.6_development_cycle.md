---
title: Some numbers from the 4.6 development cycle
url: https://lwn.net/Articles/686697/
date: "May 11, 2016"
category: "Releases-4.6"
author: "By Jonathan Corbet May 11, 2016"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 11, 2016

With the [4.6-rc7](<https://lwn.net/Articles/686660/>) release on May 8, the 4.6 kernel is nearly ready for release. It wouldn't be a proper development cycle without a set of statistics showing where the code came from this time around. With the 4.6 kernel we have seen some old names return to the lists of the top contributors, but, for the most part, it looks like a typical — if busy — cycle. 

The 4.6 merge window, with 12,172 non-merge commits, was the busiest in the history of the kernel project. By current appearances, though, 4.6 will not be setting the record for the busiest development cycle overall. The 13,354 commits merged up to 4.6-rc7 are a lot, but it seems unlikely that the total when 4.6-final comes out will exceed the 13,694 merged for 4.2, much less the 13,722 merged for 3.15. So the record set for 3.15 will, at the beginning of June, have held for a full two years. 

Those commits were contributed by 1,661 developers, which _is_ a new record, beating the 1,625 seen contributing to 4.3. 283 of those developers made their first commit to the kernel during this development cycle. The most active 4.6 developers were: 

> Most active 4.6 developers  
> ---  
> | By changesets  
> ---  
> Arnd Bergmann| 204| 1.5%  
> Chaehyun Lim| 197| 1.5%  
> Oleg Drokin| 165| 1.2%  
> Jes Sorensen| 131| 1.0%  
> Amitoj Kaur Chawla| 117| 0.9%  
> Christian König| 107| 0.8%  
> Dennis Dalessandro| 100| 0.7%  
> Mauro Carvalho Chehab| 97| 0.7%  
> Al Viro| 93| 0.7%  
> Peter Hurley| 91| 0.7%  
> Linus Walleij| 86| 0.6%  
> Geert Uytterhoeven| 86| 0.6%  
> Laxman Dewangan| 85| 0.6%  
> Andy Lutomirski| 84| 0.6%  
> Marc Zyngier| 84| 0.6%  
> Laurent Pinchart| 84| 0.6%  
> Namhyung Kim| 81| 0.6%  
> Mike Rapoport| 81| 0.6%  
> Bhaktipriya Shridhar| 80| 0.6%  
> Tomi Valkeinen| 78| 0.6%  
> | By changed lines  
> ---  
> Faisal Latif| 28327| 4.3%  
> Dennis Dalessandro| 16034| 2.4%  
> Mike Marshall| 15434| 2.3%  
> Shraddha Barke| 14126| 2.1%  
> Oleg Drokin| 7935| 1.2%  
> Josh Poimboeuf| 7917| 1.2%  
> Andrew Duggan| 6640| 1.0%  
> John Crispin| 5914| 0.9%  
> Tomi Valkeinen| 5893| 0.9%  
> Archit Taneja| 5704| 0.9%  
> Andrew F. Davis| 5513| 0.8%  
> Jiri Pirko| 5410| 0.8%  
> Santosh Shilimkar| 5338| 0.8%  
> Alexandre Bounine| 5191| 0.8%  
> Yuval Mintz| 5054| 0.8%  
> James Simmons| 4619| 0.7%  
> Laurent Pinchart| 4534| 0.7%  
> Mauro Carvalho Chehab| 4533| 0.7%  
> Sudeep Dutt| 4387| 0.7%  
> Marc Zyngier| 4230| 0.6%  
  
Longtime kernel developer Arnd Bergmann found his way to the top of the "by changesets" column mainly through [his ongoing work](<https://lwn.net/Articles/683476/>) to eliminate warnings from the ARM kernel builds. Chaehyun Lim continues to work on the wilc1000 staging driver, Oleg Drokin did a lot of cleanups in the Lustre filesystem, Jes Sorensen continues to clean up the rtl8xxxu staging driver, and Amitoj Kaur Chawla made cleanups throughout the driver subsystem. 

On the "lines changed" side, Faisal Latif got to the top of the list with the addition of the i40iw InfiniBand driver. Dennis Dalessandro worked on various RDMA drivers, Mike Marshall added the [OrangeFS](<https://lwn.net/Articles/643165/>) filesystem, and Shraddha Barke removed some unloved drivers from the staging tree. 

The number of employers seen supporting kernel work remains steady at just over 200; this count has not changed significantly since the early 3.x days. The most active employers in this cycle were: 

> Most active 4.6 employers  
> ---  
> | By changesets  
> ---  
> Intel| 2009| 15.0%  
> (Unknown)| 1358| 10.2%  
> Red Hat| 1043| 7.8%  
> (None)| 647| 4.8%  
> Linaro| 588| 4.4%  
> Outreachy| 413| 3.1%  
> Samsung| 390| 2.9%  
> SUSE| 364| 2.7%  
> Renesas Electronics| 336| 2.5%  
> IBM| 312| 2.3%  
> AMD| 307| 2.3%  
> ARM| 253| 1.9%  
> Google| 247| 1.8%  
> (Consultant)| 238| 1.8%  
> Texas Instruments| 230| 1.7%  
> Oracle| 191| 1.4%  
> Code Aurora Forum| 175| 1.3%  
> Atmel| 166| 1.2%  
> NVidia| 159| 1.2%  
> Huawei Technologies| 147| 1.1%  
> | By lines changed  
> ---  
> Intel| 131992| 20.0%  
> (Unknown)| 43271| 6.5%  
> Red Hat| 40589| 6.1%  
> (None)| 29098| 4.4%  
> Linaro| 19259| 2.9%  
> IBM| 17455| 2.6%  
> Outreachy| 17445| 2.6%  
> Omnibond Systems| 17122| 2.6%  
> Texas Instruments| 16779| 2.5%  
> Code Aurora Forum| 14484| 2.2%  
> Renesas Electronics| 14222| 2.2%  
> Google| 14183| 2.1%  
> AMD| 13581| 2.1%  
> SUSE| 12304| 1.9%  
> Samsung| 12107| 1.8%  
> Oracle| 11441| 1.7%  
> Mellanox| 10971| 1.7%  
> ARM| 10811| 1.6%  
> Facebook| 9642| 1.5%  
> (Consultant)| 8086| 1.2%  
  
As usual, little has changed in this table. Lots of cleanup patches made their way into the staging tree as part of the application process for Outreachy internships; this time around, some of the Outreachy interns are employing Coccinelle scripts and increasing their productivity accordingly. One name that might be unfamiliar is Omnibond Systems — the company behind OrangeFS. Otherwise, it's mostly the names one usually sees in this place. 

#### Reviewing Reviewed-by

Finally, let's take a quick look at Reviewed-by counts. A developer can offer a Reviewed-by tag after having reviewed a patch in depth; these tags are considered a sign that the patches in question have been closely examined and are of high quality. They are also there to credit the developers who spend time reviewing code — an important objective, since review bandwidth is one of the community's most limited resources. In the 4.6 development kernel, a total of 3,645 Reviewed-by tags were applied to patches; the most prolific sources of them were: 

> Most active 4.6 reviewers  
> ---  
> Alex Deucher| 155| 4.3%  
> Mike Marciniszyn| 141| 3.9%  
> Ira Weiny| 133| 3.6%  
> Dennis Dalessandro| 130| 3.6%  
> Christoph Hellwig| 98| 2.7%  
> Hannes Reinecke| 91| 2.5%  
> Johannes Thumshirn| 82| 2.2%  
> Oleg Drokin| 77| 2.1%  
> Daniel Vetter| 72| 2.0%  
> Ville Syrjälä| 68| 1.9%  
> Krzysztof Kozlowski| 64| 1.8%  
> Christoffer Dall| 54| 1.5%  
> Christian König| 53| 1.5%  
> Thomas Gleixner| 49| 1.3%  
> Chris Wilson| 47| 1.3%  
> Dean Luick| 44| 1.2%  
> Maarten Lankhorst| 41| 1.1%  
> James Simmons| 40| 1.1%  
> Chunming Zhou| 37| 1.0%  
> Laurent Pinchart| 36| 1.0%  
  
This table makes it clear that different developers have different views of how Reviewed-by should be used. The top reviewer is Alex Deucher, the maintainer for the AMD graphics drivers. Alex certainly reviews a lot of patches; in this case, though, most of the patches with his Reviewed-by tags also carried his Signed-off-by tags — applied when he accepted the patches for merging. It is normally understood that maintainers should review patches before applying them; most maintainers do not add a Reviewed-by tag for that work, but Alex does. 

Out of the next three reviewers on the list, Mike Marciniszyn, Ira Weiny, and Dennis Dalessandro, only Mike appears in the `MAINTAINERS` file. They do have something in common, though, in that they all work for Intel. Patches generally come out of Intel with Reviewed-by tags from these developers (among others), often [all three together](<https://git.kernel.org/linus/17fb4f2923d7fc7ee778dedc0aa60ab6f402f56c>), applied before the code has been seen by the rest of the world. We are clearly seeing the results of some sort of internal Intel review policy at work. There is no real way to know how thorough this review process is, other than to observe that Intel contributes a lot of patches and problems are rare. 

The fifth entry in the list, Christoph Hellwig, typifies the sort of review that was envisioned when this tag was created. Nobody who has had Christoph's attention applied to a patch of theirs will have missed the fact that a serious review has taken place. The work of the other four reviewers named above may well be just as thorough and just as valuable, but, since it's not _public_ review, there is no way to know for sure. 

Of course, almost all of the review work that does take place in public still does not result in a Reviewed-by tag being applied to the patches involved. So it could be argued that this tag has never really achieved its purpose of documenting and crediting the review work that is being done. 

The overall picture created by these numbers, though, is one of a development process that continues to function like a relatively well-tuned machine. The number of contributors continues to increase, the patch flow is steady, and there do not appear to be many process-scalability issues in sight.
