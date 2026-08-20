---
title: Statistics from the 3.19 development cycle
url: https://lwn.net/Articles/631509/
date: "February 4, 2015"
category: "Releases-3.19"
author: "By Jonathan Corbet February 4, 2015"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 4, 2015

As of this writing, the [3.19-rc7](<https://lwn.net/Articles/631453/>) prepatch is out and things look on track for the final 3.19 release in the near future. So it seems about time for a look at the changes that went into this release and where they came from. Once again, it has been a busy development cycle with few real surprises. 

There have been 12,461 changesets merged into the mainline for the 3.19 release. That is up a bit from 3.18 and fits well with the general trend toward larger changeset counts in each release, as can be seen in this simple plot: 

> ![\[Changesets merged per release\]](https://static.lwn.net/images/2015/csets-merged-19.png)

If the final 3.19 release does happen on February 8, as seems likely, the development cycle will have been 63 days long — the third cycle in a row of that length. Development cycle length has been decreasing for some time, but, at this point, we may be close to the minimum time it takes to produce a production-quality kernel release. 

The changes in the 3.19 release were contributed by 1,422 developers — almost the same as the number of contributors to 3.18. The most active of these developers are: 

> Most active 3.19 developers  
> ---  
> | By changesets  
> ---  
> H Hartley Sweeten| 463| 3.7%  
> Wolfram Sang| 333| 2.7%  
> Alexander Aring| 193| 1.5%  
> Benjamin Romer| 176| 1.4%  
> Lars-Peter Clausen| 154| 1.2%  
> Malcolm Priestley| 126| 1.0%  
> Al Viro| 123| 1.0%  
> Thierry Reding| 121| 1.0%  
> Takashi Iwai| 118| 0.9%  
> Daniel Vetter| 103| 0.8%  
> Geert Uytterhoeven| 103| 0.8%  
> SF Markus Elfring| 99| 0.8%  
> Johan Hedberg| 98| 0.8%  
> Michael S. Tsirkin| 95| 0.8%  
> Hans Verkuil| 84| 0.7%  
> Peter Hurley| 82| 0.7%  
> Ville Syrjälä| 79| 0.6%  
> Damien Lespiau| 76| 0.6%  
> Fabian Frederick| 73| 0.6%  
> Steven Rostedt| 71| 0.6%  
> | By changed lines  
> ---  
> Malcolm Priestley| 31115| 4.5%  
> Jeff Kirsher| 26025| 3.7%  
> Hans Verkuil| 22602| 3.3%  
> Nicolas Ferre| 22580| 3.2%  
> Paolo Bonzini| 14706| 2.1%  
> Hante Meuleman| 13694| 2.0%  
> H Hartley Sweeten| 12445| 1.8%  
> Ley Foon Tan| 10222| 1.5%  
> Takashi Sakamoto| 8704| 1.3%  
> Thierry Reding| 8260| 1.2%  
> Philipp Zabel| 7477| 1.1%  
> Joe Perches| 7268| 1.0%  
> Daniel Vetter| 6584| 0.9%  
> Pratik Patel| 6573| 0.9%  
> Alexander Aring| 6382| 0.9%  
> Vinod Koul| 6085| 0.9%  
> Finn Thain| 5663| 0.8%  
> Daeseok Youn| 5554| 0.8%  
> Hannes Reinecke| 4951| 0.7%  
> Christoph Hellwig| 4818| 0.7%  
  
There may indeed come a day, in some distant, unimaginable future, when the Comedi drivers are done and Hartley Sweeten does not contribute the most changesets to a kernel release. But that day won't happen during the 3.19 cycle. Wolfram Sang eliminated unnecessary assignments to the `owner` field of `struct platform_driver`, Alexander Aring did a lot of work on the kernel's [802.15.4](<http://en.wikipedia.org/wiki/IEEE_802.15.4>) implementation, Benjamin Romer cleaned up the `unisys` driver in the staging tree, and Lars-Peter Clausen did a bunch of sound-driver work. 

In the "lines changed" column, Malcolm Priestley topped the list with a lot of work (including the removal of a lot of code) on the `vt6655` driver in the staging tree. Jeff Kirsher contributed three patches, one of which removed the old and unloved `bcm` driver, while Hans Verkuil, as usual, did a lot of work within the Video4Linux tree. Nicolas Ferre removed a bunch of old ARM code, and Paolo Bonzini rearranged the KVM x86 code. 

The developers working on 3.19 were supported by just over 200 companies, a typical number for recent development cycles. The list of the most active companies holds few surprises: 

> Most active 3.19 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1490| 12.0%  
> (None)| 1363| 10.9%  
> Red Hat| 1029| 8.3%  
> (Unknown)| 935| 7.5%  
> (Consultant)| 617| 5.0%  
> Vision Engraving Systems| 463| 3.7%  
> SUSE| 437| 3.5%  
> Samsung| 420| 3.4%  
> Linaro| 362| 2.9%  
> Texas Instruments| 284| 2.3%  
> Google| 270| 2.2%  
> IBM| 267| 2.1%  
> Renesas Electronics| 261| 2.1%  
> Unisys| 208| 1.7%  
> AMD| 205| 1.6%  
> Free Electrons| 205| 1.6%  
> Freescale| 202| 1.6%  
> NVidia| 153| 1.2%  
> ARM| 153| 1.2%  
> Broadcom| 145| 1.2%  
> | By lines changed  
> ---  
> Intel| 95283| 13.7%  
> (None)| 78635| 11.3%  
> Red Hat| 57378| 8.3%  
> (Unknown)| 43819| 6.3%  
> Cisco| 25757| 3.7%  
> (Consultant)| 25511| 3.7%  
> Atmel| 25239| 3.6%  
> Broadcom| 18189| 2.6%  
> Linaro| 18071| 2.6%  
> SUSE| 17898| 2.6%  
> Samsung| 16295| 2.3%  
> AMD| 16186| 2.3%  
> Google| 15152| 2.2%  
> Vision Engraving Systems| 13307| 1.9%  
> Free Electrons| 11559| 1.7%  
> Altera| 11369| 1.6%  
> IBM| 11324| 1.6%  
> Code Aurora Forum| 10632| 1.5%  
> Texas Instruments| 10377| 1.5%  
> NVidia| 9639| 1.4%  
  
The percentage of developers working on their own time is holding steady at around 11%, suggesting that the long-term decline in volunteer developers may have come to an end. 

To finish things out, it has been a while since we have taken a look at the longevity of code going into the kernel — in other words, how much of the current kernel came from each development cycle? The current picture looks like this: 

**Surviving lines of code from each development cycle** ![\[Bar chart\]](https://static.lwn.net/images/2015/linetags-3.19.png)

This chart was generated by running "`git blame`" on each file in the kernel tree and identifying the commit that created each line within the file. It starts at 2.6.13, omitting the beginning of the git era at 2.6.12 for clarity; that bar, representing the 3 million lines that have not been modified since 2.6.12, would dwarf all the others. 

The story that emerges from that chart is that recent development cycles do not dramatically overshadow earlier cycles. That suggests that, once a line of code is put in place, it tends not to change much in the following cycles. Sometimes it seems that there is a lot of churn, frequently rewriting the same code, but the truth is that most of the code in the kernel has not been changed in years. 

In other words, most kernel developers are busy adding new code, which is not a particularly surprising result. It takes a lot of new code to keep up with the demands for new features and support for new hardware. The evidence suggests that the kernel community is continuing to grow in response to those demands and the process is working reasonably well.
