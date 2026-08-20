---
title: Some numbers from the 3.2 development cycle
url: https://lwn.net/Articles/472852/
date: "December 21, 2011"
category: "Releases-3.2"
author: "By Jonathan Corbet December 21, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 21, 2011

The 3.2 kernel development cycle always had the potential to be a little different. The prolonged kernel.org outage had left a number of subsystem trees scrambling for new homes; that led to a delayed opening of the merge window. The actual merging of changes happened mostly during the Kernel Summit in Prague. And, even before the normal process got disrupted, this looked like a more than usually active cycle. Despite these challenges, the 3.2 kernel process seems to have worked pretty much as it usually does once it got started. 

As of this writing (just after the release of 3.2-rc6), some 11,655 non-merge changesets have been pulled into the mainline kernel; these changesets were contributed by 1,289 developers. At that count, 3.2 is the fourth largest development cycle ever. Chances are good that it will surpass 2.6.29 (11,678 changes) to move up to the number-three position; getting past 2.6.30 (11,989) seems harder - if not impossible - at this point, while passing 2.6.25 (12,243) to become the busiest cycle ever seems quite unlikely. If we want to set a new record for changes merged, we're going to have to try harder. 

A lot of code was removed in this cycle, so the total growth of the kernel was 176,000 lines - a relatively modest number. 

The most active developers this time around were: 

> Most active 3.2 developers  
> ---  
> | By changesets  
> ---  
> Larry Finger| 302| 2.6%  
> Paul Gortmaker| 234| 2.0%  
> Mark Brown| 226| 1.9%  
> Axel Lin| 220| 1.9%  
> K. Y. Srinivasan| 165| 1.4%  
> Jonathan Cameron| 159| 1.4%  
> Roland Vossen| 157| 1.3%  
> Ben Skeggs| 121| 1.0%  
> Dmitry Eremin-Solenikov| 117| 1.0%  
> Christoph Hellwig| 113| 1.0%  
> Nicolas Pitre| 109| 0.9%  
> Al Viro| 104| 0.9%  
> Dan Carpenter| 101| 0.9%  
> Arend van Spriel| 100| 0.9%  
> Mark Einon| 99| 0.8%  
> Guennadi Liakhovetski| 98| 0.8%  
> Laurent Pinchart| 95| 0.8%  
> Takashi Iwai| 92| 0.8%  
> Johannes Berg| 91| 0.8%  
> J. Bruce Fields| 88| 0.8%  
> | By changed lines  
> ---  
> Arend van Spriel| 105436| 9.2%  
> Kalle Valo| 100542| 8.8%  
> Larry Finger| 84036| 7.3%  
> Roland Vossen| 34944| 3.1%  
> Edwin Rong| 21876| 1.9%  
> Mark Brown| 13771| 1.2%  
> Mark Einon| 13597| 1.2%  
> Richard Kuo| 12223| 1.1%  
> Rasesh Mody| 11792| 1.0%  
> Joe Thornber| 10000| 0.9%  
> Jonathan Cameron| 9776| 0.9%  
> Kukjin Kim| 8920| 0.8%  
> Franky (Zhenhui) Lin| 8383| 0.7%  
> Linus Walleij| 7317| 0.6%  
> Emmanuel Grumbach| 6838| 0.6%  
> Felipe Balbi| 6783| 0.6%  
> David Kilroy| 6356| 0.6%  
> Takashi Iwai| 6188| 0.5%  
> Shawn Guo| 6021| 0.5%  
> Jeff Kirsher| 6015| 0.5%  
  
Larry Finger put a vast amount of work into cleaning up the rtl8192e driver in the staging tree, making it quite a bit smaller in the process. Paul Gortmaker split the `EXPORT_SYMBOL*` macros into `<linux/export.h>`; after that, many files no longer needed to include `<linux/module.h>`. The real advantage of that kind of work, beyond minimizing the interactions between various parts of the kernel, is that it makes the kernel compilation process faster. Mark Brown, as usual, wrote or improved vast numbers of audio drivers. Axel Lin did a lot of cleanup work, mostly in the audio driver subsystem, while K. Y. Srinivasan continued the seemingly unending task of getting Microsoft's "hv" drivers ready to move into the mainline. 

Arend van Spriel topped the list of "lines changed" by moving the brcm80211 driver from staging into the mainline tree. One could argue that this change should be accounted as a rename (which doesn't change any lines), but it does not show up that way in the source history: one patch added the drivers to mainline, while a separate patch removed them from staging. Kalle Valo removed the ath6kl driver from staging, since support for this hardware had been added to the mainline "ath" driver; as a result, he topped the list of developers who removed the most code from the kernel. Larry Finger's work has already been mentioned. Roland Vossen worked hard on the brcm80211 cleanup, and Edwin Rong added a driver for the Realtek RTS5139 cardreader to the staging tree. 

The top five entries in the "lines changed" column are all thus related to the staging tree. Some have argued in the past that staging should be excluded from these statistics. There is a valid point behind those arguments, but it should also be noted that much of the activity this time was around movement of code from staging into the mainline. That suggests that staging is working the way it was intended to, and that work done there benefits the mainline in the end. 

191 employers were identified as having supported work on the 3.2 kernel. Among those, the most active were: 

> Most active 3.2 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1722| 14.8%  
> Red Hat| 988| 8.5%  
> (Unknown)| 863| 7.4%  
> Intel| 844| 7.2%  
> Broadcom| 493| 4.2%  
> Texas Instruments| 482| 4.1%  
> IBM| 412| 3.5%  
> Novell| 347| 3.0%  
> Wind River| 281| 2.4%  
> Qualcomm| 251| 2.2%  
> Wolfson Micro| 248| 2.1%  
> Samsung| 232| 2.0%  
> MiTAC| 220| 1.9%  
> (Consultant)| 208| 1.8%  
> Nokia| 202| 1.7%  
> Linaro| 202| 1.7%  
> Oracle| 189| 1.6%  
> Freescale| 182| 1.6%  
> Google| 182| 1.6%  
> Microsoft| 177| 1.5%  
> | By lines changed  
> ---  
> Broadcom| 256549| 22.4%  
> (None)| 202387| 17.7%  
> Qualcomm| 133277| 11.6%  
> Red Hat| 48673| 4.2%  
> (Unknown)| 43254| 3.8%  
> Intel| 43094| 3.8%  
> Texas Instruments| 31529| 2.8%  
> Samsung| 30233| 2.6%  
> IBM| 22279| 1.9%  
> Realsil Micro| 22065| 1.9%  
> Brocade| 21734| 1.9%  
> Freescale| 16657| 1.5%  
> Wolfson Micro| 16217| 1.4%  
> ST Ericsson| 14334| 1.3%  
> Novell| 14161| 1.2%  
> Code Aurora Forum| 13706| 1.2%  
> Univ. of Cambridge| 12350| 1.1%  
> Linaro| 10708| 0.9%  
> (Consultant)| 9263| 0.8%  
> Marvell| 8640| 0.8%  
  
Red Hat remains the top corporate submitter of patches to the kernel, but its lead looks less commanding than it once was. Meanwhile, companies like Texas Instruments and Samsung continue to increase their contributions to the kernel - embedded systems vendors are now a huge part of the development community. There also seems to be an increase in the amount of code coming from industry consortia like Linaro - again, mostly focused in the embedded area. But, with over 190 companies participating, we clearly still have interest from beyond just the embedded realm. 

As of this writing, the 3.2 kernel looks likely to be released right around the end of the year, after one more -rc release. If that schedule holds, this cycle will have required less than 70 days, significantly shorter than the average (which is about 80 days) despite the large volume of changes. The process, in other words, appears to be working fairly well despite the kernel.org difficulties and the delayed start. Sooner or later, we are bound to run into a problem that will throw a significant wrench into the works - life is just like that - but that certainly hasn't happened this time around.
