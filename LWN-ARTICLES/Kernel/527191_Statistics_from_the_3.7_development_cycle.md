---
title: Statistics from the 3.7 development cycle
url: https://lwn.net/Articles/527191/
date: "November 28, 2012"
category: "Releases-3.7"
author: "By Jonathan Corbet November 28, 2012"
---

By **Jonathan Corbet**  
November 28, 2012

The [3.7-rc7](<https://lwn.net/Articles/526911/>) prepatch came out on November 25; it may well be the last prepatch for the 3.7 development cycle. 3.7 was one of the more active cycles in recent history, with nearly 12,000 non-merge changesets incorporated by the time of this writing. It's time for our traditional look at what was done during this cycle and where all that work came from. 

The 3.7 merge window was especially busy this time around. Here are some counts for recent kernels: 

> Kernel| -rc1| Total  
> ---|---|---  
> 3.0| 7,333| 9,153  
> 3.1| 7,202| 8,693  
> 3.2| 10,214| 11,881  
> 3.3| 8,899| 10,550  
> 3.4| 9,249| 10,899  
> 3.5| 9,534| 10,957  
> 3.6| 8,587| 10,247  
> 3.7| 10,409| 11,815  
  
The 3.7 development cycle, thus, saw the most active merge window in the 3.x era; it is, in fact, the most active merge window ever. Even allowing for the fact that 3.7 will add a few more changesets before final release, the 2.6.25 kernel, at 12,243 changesets total, will probably still hold the record for the most active development cycle ever, but the 2.6.25 merge window only saw 9,450 changesets merged. One could conclude from these numbers that we are getting better at getting our changes in during the merge window — and at having fewer things to fix thereafter. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Nearly 395,000 lines of code were removed from the kernel this time around. That must be balanced against the 719,000 lines that were added, though; the kernel grew by almost 324,000 lines as a result. 

1,271 developers contributed to the 3.7 kernel — a relatively high number, but not out of line with previous development cycles. The lists of the most active developers do see some changes this time around, though: 

> Most active 3.7 developers  
> ---  
> | By changesets  
> ---  
> H Hartley Sweeten| 417| 3.5%  
> Antti Palosaari| 216| 1.8%  
> Al Viro| 167| 1.4%  
> Wei Yongjun| 145| 1.2%  
> Sachin Kamat| 138| 1.2%  
> Mark Brown| 136| 1.2%  
> Eric W. Biederman| 130| 1.1%  
> Daniel Vetter| 122| 1.0%  
> David Howells| 119| 1.0%  
> Hans Verkuil| 119| 1.0%  
> Greg Kroah-Hartman| 116| 1.0%  
> Arnd Bergmann| 112| 0.9%  
> Peter Senna Tschudin| 104| 0.9%  
> Ben Skeggs| 97| 0.8%  
> Peter Ujfalusi| 96| 0.8%  
> Ian Abbott| 96| 0.8%  
> Devendra Naga| 90| 0.8%  
> David S. Miller| 84| 0.7%  
> Takashi Iwai| 83| 0.7%  
> Johannes Berg| 78| 0.7%  
> | By changed lines  
> ---  
> David Howells| 65206| 7.6%  
> Ben Skeggs| 50282| 5.8%  
> David Daney| 46825| 5.4%  
> Arnd Bergmann| 17505| 2.0%  
> Sebastian Andrzej Siewior| 16088| 1.9%  
> Daniel Cotey| 14157| 1.6%  
> H Hartley Sweeten| 13566| 1.6%  
> Catalin Marinas| 13519| 1.6%  
> Antti Palosaari| 12336| 1.4%  
> Bill Pemberton| 10935| 1.3%  
> Dan Magenheimer| 10509| 1.2%  
> Ezequiel Garcia| 10211| 1.2%  
> David S. Miller| 9258| 1.1%  
> Hans Verkuil| 8686| 1.0%  
> Will Deacon| 8404| 1.0%  
> Shawn Guo| 7464| 0.9%  
> Alois Schlögl| 7301| 0.8%  
> Roland Stigge| 6987| 0.8%  
> Greg Kroah-Hartman| 6920| 0.8%  
> Laurent Pinchart| 6107| 0.7%  
  
In a repeat of his 3.6 performance, H. Hartley Sweeten hit the top of the by-changesets list with a vast number of patches preparing the comedi drivers for graduation from the staging tree (removing over 5000 lines of code in the process). Antti Palosaari did a lot of work on drivers in the Video4Linux2 subsystem. Al Viro continues to refactor and clean up the VFS and core kernel areas with some excursions into most architecture subtrees. Wei Yongjun and Sachin Kamat both did a lot of cleanup work all over the driver tree. 

David Howells ended up at the top of the "lines changed" column mostly by virtue of the [user-space API header file thrashup](<https://lwn.net/Articles/507794/>), but he also contributed code for module signing and more. Ben Skeggs merged a major reworking of the nouveau driver, David Daney improved support for MIPS OCTEON processors, Arnd Bergmann's many patches were dominated by the removal of the unused mach-bcmring architecture code, and Sebastian Andrzej Siewior did a lot of work on the USB gadget driver subsystem. 

Worth noting in passing: Fengguang Wu is credited with 63 bug reports during this cycle, almost 11% of the total. The others with at least ten reports are Dan Carpenter (21), Randy Dunlap (16), Stephen Rothwell (15), Paul McKenney (11), and Alex Lyakas (10). Every one of those reports resulted in a bug that was fixed before this kernel was released in stable form. 

An even 200 employers (that we know about) contributed during the 3.7 cycle. The most active of these were: 

> Most active 3.7 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1435| 12.1%  
> Red Hat| 1159| 9.8%  
> (Unknown)| 843| 7.1%  
> Intel| 800| 6.8%  
> Texas Instruments| 597| 5.1%  
> IBM| 516| 4.4%  
> Linaro| 509| 4.3%  
> Vision Engraving Systems| 417| 3.5%  
> SUSE| 356| 3.0%  
> Google| 245| 2.1%  
> Samsung| 198| 1.7%  
> Freescale| 181| 1.5%  
> Oracle| 177| 1.5%  
> Wolfson Microelectronics| 148| 1.3%  
> AMD| 144| 1.2%  
> Trend Micro| 144| 1.2%  
> Cisco| 138| 1.2%  
> Linux Foundation| 132| 1.1%  
> Arista Networks| 130| 1.1%  
> NVIDIA| 123| 1.0%  
> | By lines changed  
> ---  
> Red Hat| 157023| 18.2%  
> (None)| 80191| 9.3%  
> (Unknown)| 71992| 8.3%  
> Cavium| 46757| 5.4%  
> IBM| 39227| 4.5%  
> Intel| 33381| 3.9%  
> Linaro| 28900| 3.4%  
> Texas Instruments| 28493| 3.3%  
> ARM| 24913| 2.9%  
> Oracle| 24095| 2.8%  
> NVIDIA| 19167| 2.2%  
> linutronix| 17211| 2.0%  
> Vision Engraving Systems| 14844| 1.7%  
> Samsung| 14519| 1.7%  
> Microtrol S.R.L.| 12800| 1.5%  
> PHILOSYS Software| 10311| 1.2%  
> SUSE| 10226| 1.2%  
> Marvell| 10067| 1.2%  
> Cisco| 9828| 1.1%  
> Pengutronix| 9793| 1.1%  
  
There are few surprises here. Texas Instruments has reached a new high in its contribution volume, a trend which, unfortunately, may not continue after the [recent changes](<http://news.cnet.com/8301-1001_3-57549995-92/texas-instruments-exits-consumer-phones-lays-off-1700/>) play out there. AMD, too, [seems unlikely](<http://www.theregister.co.uk/2012/11/07/amd_closes_german_osrc_lab/>) to remain on this list in the future. Meanwhile Red Hat maintains its place at the top of the list, where it has been since we first started generating these statistics. 

And that is where things stand as the 3.7 kernel approaches its final release. Things appear to be running smoothly, with most development cycles taking less than 70 days to complete (if there is no 3.7-rc8, this cycle will run about 64 days). Stay tuned for the about-to-begin 3.8 cycle, with a release to be expected in early February, 2013.
