---
title: A look at the 3.12 development cycle
url: https://lwn.net/Articles/570483/
date: "October 23, 2013"
category: "Releases-3.12"
author: "By Jonathan Corbet October 23, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 23, 2013

As of this writing, the [3.12-rc6](<https://lwn.net/Articles/570990/>) prepatch has been released, Linus seems happy with the state of the kernel, and, in general, there are few reports of problems on the mailing lists. If things continue to stabilize, the 3.12 cycle might be a short one, even by recent standards. So, clearly, it's time to get the traditional development statistics article out there. 

There have been 10,480 non-merge changesets pulled into the mainline repository during this cycle (so far). That means that 3.12 may be the slowest cycle since 3.6 which, almost exactly one year ago, came out with 10,247 changesets. An unscientific look at recent release history suggests that kernels released in the (northern hemisphere) fall tend to include fewer changesets than those released at other times of the year, possibly reflecting lower productivity while developers take time off over the summer. 

That said, 10,480 changesets is still quite a bit of work. Those changesets were contributed by 1,259 developers (a typical number for recent kernels), added 601,000 lines of code, and removed 279,000 for a net gain of 322,000 lines. The most active developers in this cycle were: 

> Most active 3.12 developers  
> ---  
> | By changesets  
> ---  
> Sachin Kamat| 261| 2.5%  
> Jingoo Han| 241| 2.3%  
> Mark Brown| 209| 2.0%  
> Greg Kroah-Hartman| 197| 1.9%  
> H Hartley Sweeten| 160| 1.5%  
> Alex Deucher| 151| 1.4%  
> Laurent Pinchart| 140| 1.3%  
> Daniel Vetter| 138| 1.3%  
> Fabio Estevam| 114| 1.1%  
> Chris Metcalf| 103| 1.0%  
> Dan Carpenter| 96| 0.9%  
> Dave Chinner| 90| 0.9%  
> Peter Hurley| 83| 0.8%  
> Joe Perches| 80| 0.8%  
> Ben Hutchings| 77| 0.7%  
> Magnus Damm| 76| 0.7%  
> Rafael J. Wysocki| 73| 0.7%  
> Lars-Peter Clausen| 73| 0.7%  
> Trond Myklebust| 67| 0.6%  
> Axel Lin| 65| 0.6%  
> | By changed lines  
> ---  
> Larry Finger| 92908| 12.6%  
> Jesse Brandeburg| 30520| 4.2%  
> Greg Kroah-Hartman| 29740| 4.0%  
> H Hartley Sweeten| 25932| 3.5%  
> Alex Deucher| 18026| 2.5%  
> Ben Hutchings| 17660| 2.4%  
> Rob Clark| 15703| 2.1%  
> Bradley Grove| 15687| 2.1%  
> Dave Chinner| 15099| 2.1%  
> Scott Kilau| 14712| 2.0%  
> Lidza Louina| 13474| 1.8%  
> Laurent Pinchart| 11676| 1.6%  
> Rajendra Nayak| 10866| 1.5%  
> Chris Metcalf| 8924| 1.2%  
> Tomi Valkeinen| 8881| 1.2%  
> Feng-Hsin Chiang| 6813| 0.9%  
> Yuan-Hsin Chen| 6813| 0.9%  
> Ambresh K| 5528| 0.8%  
> Hans Verkuil| 5385| 0.7%  
> Atul Deshmukh| 4849| 0.7%  
  
Sachin Kamat and Jingoo Han both contributed a wide range of cleanup patches all over the driver subsystem. Mark Brown continues to do substantial work in the sound, SPI, and regulator driver subsystems, among others. Greg Kroah-Hartman integrated a number of low-level device model changes, along with Lustre filesystem fixups and more. H. Hartley Sweeten's crusade to clean up the Comedi drivers continues; that work resulted in the removal of over 21,000 lines of code this time around. 

On the "lines changed" side, Larry Finger added the Realtek RTL8188EU wireless network driver to the staging tree. Jesse Brandeburg added the Intel i40e network driver. Greg's and Hartley's work was, in both cases, dominated by the removal of large amounts of unneeded driver code, while Alex Deucher continues to add functionality to the Radeon driver. 

A total of 212 employers (that we know of) supported work on the 3.12 release. The most active of those were: 

> Most active 3.12 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1028| 9.8%  
> (None)| 964| 9.2%  
> Linaro| 732| 7.0%  
> Red Hat| 707| 6.7%  
> (Unknown)| 609| 5.8%  
> Samsung| 492| 4.7%  
> IBM| 390| 3.7%  
> Freescale| 256| 2.4%  
> Renesas Electronics| 249| 2.4%  
> Texas Instruments| 245| 2.3%  
> Linux Foundation| 225| 2.1%  
> SUSE| 206| 2.0%  
> Oracle| 183| 1.7%  
> Free Electrons| 183| 1.7%  
> (Consultant)| 182| 1.7%  
> AMD| 178| 1.7%  
> Vision Engraving| 175| 1.7%  
> Google| 161| 1.5%  
> Huawei Technologies| 124| 1.2%  
> Broadcom| 120| 1.1%  
> | By lines changed  
> ---  
> (None)| 134134| 18.3%  
> Intel| 61227| 8.3%  
> Red Hat| 49820| 6.8%  
> Linux Foundation| 32955| 4.5%  
> Vision Engraving| 26848| 3.7%  
> Linaro| 26081| 3.5%  
> Texas Instruments| 24518| 3.3%  
> AMD| 24389| 3.3%  
> (Unknown)| 22927| 3.1%  
> Renesas Electronics| 20656| 2.8%  
> Outreach Program for Women| 19649| 2.7%  
> Solarflare Comm.| 18303| 2.5%  
> ATTO Technology| 15688| 2.1%  
> Digi International| 14720| 2.0%  
> Faraday Technology| 13626| 1.9%  
> IBM| 13554| 1.8%  
> Samsung| 13083| 1.8%  
> Tilera| 12256| 1.7%  
> Freescale| 12104| 1.6%  
> SUSE| 11265| 1.5%  
  
This is not the first time that Red Hat has been upstaged as the top corporate contributor, but it has never been as low as #4. Linaro, instead, continues to increase its contributions to the kernel, as do a number of mobile and embedded companies. The number of changes from volunteers is down slightly, in keeping with the steady trend over the last few years. Developers brought in through the Outreach Program for Women continued to contribute significantly during this cycle. 

Your editor is often asked to summarize the origin of kernel patches geographically — how many are coming from $COUNTRY? That question can be hard to answer. But there is another question that is a bit easier: every commit in the repository has a time stamp, and that time stamp includes a time zone. It is a relatively easy matter to pass over a range of commits and summarize which time zones appear most often. 

![\[Time
zones plot\]](https://static.lwn.net/images/2013/3.12-zones.png) The result of this work appears in the plot to the right. One should bear in mind that this data is necessarily somewhat noisy; there is nothing that constrains developers' machines to have their time set in the zone where they physically reside. Daylight saving time can also add noise to the picture. In the aggregate, though, there are some interesting things to be seen here. 

Starting at the top, +10 includes parts of Russia, parts of Indonesia, and, most importantly for this study, parts of Australia. The +9 zone, instead, is mainly Japan and Korea, while +8 is Western Australia and China. About 15% of the changes to 3.12 came from those three time zones. 

There is only one country that lives in +5:30 — India. The number of contributions from India has been growing for a while; it's now 6% of the total. Going west from there, +2 to +4 will be dominated by continental Europe, with the central European time zone accounting for 23% of the changes in 3.12. The UK and Ireland, at +1, put in another 10%. 

The western hemisphere (negative) zones will be dominated by North America. The -3 zone, however, only covers Newfoundland and Labrador in the northern hemisphere; the relative scarcity of kernel hackers in that part of the world leads one to conclude that 5% of the patches for 3.12 came from Brazil and Argentina, both of which reside in that zone. Your editor's time zone (-6), alas, was the source of only 1% of the changes going into this release; it must be time to pull together some white-space patches to improve that situation. 

To state things more generally: one could say that Asia and Australia contributed 22% of the changes to 3.12, while Europe contributed 43%, North America 30%, and South America 5%. These numbers are clearly approximate, but they probably do not hugely distort the reality. The Linux kernel project truly is global in scope, with developers representing much of the planet participating. All told, it has the appearance of a healthy and thriving community.
