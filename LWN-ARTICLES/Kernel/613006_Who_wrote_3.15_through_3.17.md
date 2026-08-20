---
title: Who wrote 3.15 through 3.17
url: https://lwn.net/Articles/613006/
date: "September 24, 2014"
category: "Releases-3.15; Releases-3.16; Releases-3.17"
author: "By Jonathan Corbet September 24, 2014"
---

By **Jonathan Corbet**  
September 24, 2014

When writing up the [3.14 development statistics](<https://lwn.net/Articles/590354/>), your editor publicly wondered if compiling those reports for every development cycle made sense. That question was followed by a bit of a break; there were no "who wrote..." articles for the 3.15 or 3.16 development cycles. Now that just over six months have passed and the 3.17 kernel is nearing release, it seems like it may be time to take another look at how the kernel development process is working. 

Since 3.14, kernel release activity has looked like this: 

> Version | Date | CSets | Devs | Lines (thousands)  
> ---|---|---|---|---  
> Added | Removed | Delta  
> 3.15 | Jun 8 | 13,722 | 1492 | 1066 | 707 | 360  
> 3.16 | Aug 3 | 12,804 | 1478 | 578 | 329 | 249  
> 3.17 | Sep 28* | 12,153 | 1408 | 692 | 708 | -16  
> 3.15-17 |  | 38,679 | 2546 | 2336 | 1744 | 593  
  
A few interesting things jump out of these numbers. The 3.12 cycle had contributions from 1257 developers. By 3.13, that had increased to 1339, and 3.14 had patches from exactly 1400 developers. So the count of developers contributing to each kernel release, which had hovered in the 1200's for some time, has shown a significant increase. The active kernel development community continues to grow. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The kernel itself also continues to grow, but 3.17 looks like a rare exception. Thanks to the removal of a bunch of unloved code from the staging tree, 3.17 is actually smaller than its predecessor. That has only happened one other time in the history of the Linux kernel; 2.6.36 was smaller than 2.6.35 thanks to the removal of a pile of defconfig files. The overall trend remains unchanged, though; the kernel grew by almost 600,000 lines in the last three releases. 

As of [3.17-rc6](<https://lwn.net/Articles/612743/>), Linus was thinking that he would be able to do the 3.17 final release on September 28\. Should that schedule hold, the 3.17 kernel will have been produced in a mere 56 days — as was 3.16. Your editor has remarked on the trend of the shortening kernel release cycle for a while. Here is what that trend looks like now (again, assuming the 3.17 release is not delayed): 

> ![\[Development cycle length chart\]](https://static.lwn.net/images/2014/dev-cycle-length.png)

So the kernel development cycle, it seems, continues to get shorter. How much longer that trend can continue is unclear, though; there must be a minimum period required to get a high-quality release together. One other potentially interesting point: it should be remembered that the final stabilization of the 3.15 release overlapped with the 3.16 merge window. That probably had little to do with why the 3.15 cycle took longer than many others; it was the result of some difficult-to-find last-minute bugs. But one could argue that the 3.16 development cycle should really be counted as being one week longer than the release dates would indicate. 

#### Contributors

As can be seen from the table above, 38,679 non-merge changesets were pulled into the mainline repository for the 3.15-3.17 development cycles. Of the 2546 developers who contributed changes during this time, the most active were: 

> Most active developers, 3.15-3.17  
> ---  
> | By changesets  
> ---  
> Hartley Sweeten| 919| 2.4%  
> Jes Sorensen| 767| 2.0%  
> Malcolm Priestley| 544| 1.4%  
> Fabian Frederick| 382| 1.0%  
> Navin Patidar| 378| 1.0%  
> Laurent Pinchart| 330| 0.9%  
> Sachin Kamat| 327| 0.8%  
> Russell King| 316| 0.8%  
> Axel Lin| 301| 0.8%  
> Johan Hedberg| 300| 0.8%  
> Geert Uytterhoeven| 296| 0.8%  
> Daniel Vetter| 278| 0.7%  
> Takashi Iwai| 275| 0.7%  
> Jingoo Han| 265| 0.7%  
> Thomas Gleixner| 260| 0.7%  
> Alexander Shiyan| 240| 0.6%  
> Ville Syrjälä| 235| 0.6%  
> Joe Perches| 233| 0.6%  
> Tejun Heo| 231| 0.6%  
> Lars-Peter Clausen| 226| 0.6%  
> | By changed lines  
> ---  
> Tomi Valkeinen| 318894| 10.9%  
> Kristina Martšenko| 165102| 5.6%  
> Larry Finger| 164869| 5.6%  
> Andrzej Pietrasiewicz| 108036| 3.7%  
> Mauro Carvalho Chehab| 71253| 2.4%  
> Greg Kroah-Hartman| 68260| 2.3%  
> Dave Chinner| 48267| 1.6%  
> Devin Heitmueller| 46125| 1.6%  
> Malcolm Priestley| 35231| 1.2%  
> Jes Sorensen| 29412| 1.0%  
> Navin Patidar| 28871| 1.0%  
> Hans Verkuil| 27813| 0.9%  
> Ben Skeggs| 26293| 0.9%  
> Mark Hounschell| 24285| 0.8%  
> Ken Cox| 23213| 0.8%  
> Hartley Sweeten| 21246| 0.7%  
> Jason Cooper| 20344| 0.7%  
> Linus Walleij| 19898| 0.7%  
> Jake Edge| 18218| 0.6%  
> Maxime Ripard| 14669| 0.5%  
  
As is usually the case, Hartley Sweeten contributed more changesets than any other developer; all of those were against the COMEDI drivers in the staging tree. All told, nearly 6,000 patches have been applied against just that subsystem since its entry into staging. Jes Sorensen's work was nearly all against the rtl8723au driver, while Malcolm Priestly worked on the vt6656 driver; both of those drivers are also in the staging tree. Fabian Frederick contributed cleanups throughout the kernel tree, while Navin Patidar focused on the rtl8188eu driver which, unsurprisingly, is also in the staging tree. 

In the "lines changed" column, Tomi Valkeinen reached the top with extensive work on the ARM OMAP architecture code and related device tree files. Kristina Martšenko removed 14 drivers from the staging tree, making her the developer who removed the most code during this time. Larry Finger continues his work to rationalize the Realtek wireless drivers in the staging tree, Andrzej Pietrasiewicz did a lot of work in the USB gadget driver, and Video4Linux subsystem maintainer Mauro Carvalho Chehab did extensive work throughout that tree. 

The 3.15-3.17 development cycles saw contributions from at least 312 employers, the most active of whom were: 

> Most active employers, 3.15-3.17  
> ---  
> | By changesets  
> ---  
> (None)| 4492| 11.6%  
> Intel| 4088| 10.6%  
> Red Hat| 3577| 9.2%  
> (Unknown)| 3409| 8.8%  
> Linaro| 1702| 4.4%  
> Samsung| 1646| 4.3%  
> SUSE| 1243| 3.2%  
> IBM| 1050| 2.7%  
> (Consultant)| 1016| 2.6%  
> Texas Instruments| 942| 2.4%  
> Vision Engraving Systems| 919| 2.4%  
> Google| 763| 2.0%  
> Renesas Electronics| 753| 1.9%  
> Free Electrons| 753| 1.9%  
> Freescale| 620| 1.6%  
> C-DAC| 400| 1.0%  
> Oracle| 390| 1.0%  
> Imagination Technologies| 361| 0.9%  
> NVidia| 355| 0.9%  
> FOSS Outreach Program for Women| 336| 0.9%  
> | By lines changed  
> ---  
> (None)| 408176| 13.9%  
> Texas Instruments| 357058| 12.2%  
> (Unknown)| 338760| 11.6%  
> Red Hat| 259264| 8.8%  
> Samsung| 249613| 8.5%  
> Intel| 180869| 6.2%  
> Linaro| 93125| 3.2%  
> Linux Foundation| 68988| 2.4%  
> SUSE| 52213| 1.8%  
> (Consultant)| 45952| 1.6%  
> IBM| 44809| 1.5%  
> Free Electrons| 42917| 1.5%  
> Cisco| 33254| 1.1%  
> Freescale| 32636| 1.1%  
> C-DAC| 30405| 1.0%  
> Renesas Electronics| 29973| 1.0%  
> Google| 29957| 1.0%  
> Realtek| 27888| 1.0%  
> NVidia| 27232| 0.9%  
> COMPRO Intelligent Solutions| 24722| 0.8%  
  
As usual, this picture has remained relatively stable from one release to the next. Mildly notable is the increase in contributions from developers working on their own time, though it would be hard to say that the long-term trend toward decreasing volunteer contributions has ended at this point. 

#### Reviews and conclusion

Finally, it can be interesting to look at who is attaching `Reviewed-by` tags to patches. That tag is meant both as an indicator that the patch has been reviewed and a means for crediting developers who perform those reviews. The developers with the most `Reviewed-by` tags during this period were: 

> Developers with the most Reviewed-by tags  
> ---  
> Ian Abbott| 766| 11.0%  
> Josh Triplett| 207| 3.0%  
> Tomasz Figa| 155| 2.2%  
> Christoph Hellwig| 142| 2.0%  
> Ville Syrjälä| 132| 1.9%  
> Chris Wilson| 123| 1.8%  
> Johannes Berg| 122| 1.8%  
> Jesse Barnes| 103| 1.5%  
> Guenter Roeck| 98| 1.4%  
> Pieter-Paul Giesberts| 92| 1.3%  
> David Herrmann| 87| 1.3%  
> Dave Chinner| 86| 1.2%  
> Hartley Sweeten| 86| 1.2%  
> Imre Deak| 84| 1.2%  
> Alex Elder| 84| 1.2%  
> Rodrigo Vivi| 80| 1.2%  
> Alex Deucher| 74| 1.1%  
> Damien Lespiau| 73| 1.1%  
> Daniel (Deognyoun) Kim| 71| 1.0%  
> Franky (Zhenhui) Lin| 66| 1.0%  
  
Ian Abbott, it seems, has reviewed 766 patches in the 182 days covered by these three development cycles — just over four patches every day, with no breaks for weekends or holidays. It turns out that almost all of those patches were the COMEDI changes submitted by Hartley Sweeten. Josh Triplett, instead, reviewed a wide range of changes from many developers; most of those were changes related to or involving read-copy-update. Tomasz Figa concerns himself with ARM-related changes, Christoph Hellwig is a longstanding reviewer of storage- and filesystem-related patches, and reviews changes to DRM (graphics) drivers. 

What is not reflected here, of course, is the vast amount of patch review work that never results in a `Reviewed-by` tag. In fact, your editor would assert that this mechanism is not working as intended at this point. It is failing to document the bulk of the review work that is being done and serves mostly to highlight which developers make the effort to offer an explicit `Reviewed-by` tag. 

To summarize: what has changed in the six months since LWN last published a set of development statistics? The answer is "not much." The kernel development process continues to roll along, producing releases in a fairly predictable schedule. The pace continues to increase, the community continues to grow, and the development cycle continues to shorten. These are all trends that we have seen for a while, so, to a great extent, it all looks like business as usual.
