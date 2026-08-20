---
title: Who wrote 2.6.37
url: https://lwn.net/Articles/420658/
date: "December 30, 2010"
category: "Releases-2.6.37"
author: "By Jonathan Corbet December 30, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 30, 2010

The 2.6.37 development cycle is coming to a close; that must mean that it is time to look at the development statistics for this release. 2.6.37 has been a more active cycle than most, with 11,220 non-merge changesets added (as of 2.6.37-rc7). Only three cycles have seen more changes: 2.6.25 (12,243 changesets), 2.6.29 (11,678), and 2.6.30 (11,989). The relatively slow period since 2.6.33 appears to have come to an end. 

The 2.6.36 kernel was unique in that it was actually smaller than its predecessor. 2.6.37 does not continue that trend; some 1,140,000 lines of code were added, and 641,000 lines were removed, for a net growth of 494,000 lines. Most notably, perhaps: the 2.6.37 kernel includes patches from 1,250 developers, the highest ever. The development community, clearly, is active and growing. 

The most active contributors this time around were: 

> Most active 2.6.37 developers  
> ---  
> | By changesets  
> ---  
> Chris Wilson| 211| 1.9%  
> Greg Kroah-Hartman| 171| 1.5%  
> Eric Dumazet| 166| 1.5%  
> Johannes Berg| 149| 1.3%  
> Thomas Gleixner| 147| 1.3%  
> Paul Mundt| 140| 1.2%  
> Mauro Carvalho Chehab| 122| 1.1%  
> Joe Perches| 107| 1.0%  
> Avi Kivity| 105| 0.9%  
> Mark Brown| 100| 0.9%  
> Hans Verkuil| 100| 0.9%  
> Namhyung Kim| 100| 0.9%  
> Dan Carpenter| 99| 0.9%  
> Christoph Hellwig| 93| 0.8%  
> Jean Delvare| 88| 0.8%  
> Axel Lin| 88| 0.8%  
> Daniel Vetter| 87| 0.8%  
> Vasiliy Kulikov| 86| 0.8%  
> Arnd Bergmann| 86| 0.8%  
> Julia Lawall| 84| 0.7%  
> | By changed lines  
> ---  
> Henry Ptasinski| 143303| 9.6%  
> Greg Kroah-Hartman| 100861| 6.7%  
> Vipin Mehta| 92398| 6.2%  
> Luis R. Rodriguez| 71636| 4.8%  
> Mark Brown| 53690| 3.6%  
> David Cross| 48198| 3.2%  
> Dmitry Kravkov| 47198| 3.1%  
> Larry Finger| 40378| 2.7%  
> Krishna Gudipati| 38712| 2.6%  
> Stefan Richter| 29434| 2.0%  
> Stephen Hemminger| 28504| 1.9%  
> Rasesh Mody| 22335| 1.5%  
> Prashant P. Shah| 15086| 1.0%  
> Michael Chan| 14118| 0.9%  
> Christian Lamparter| 13530| 0.9%  
> Liam Girdwood| 13324| 0.9%  
> William Hubbs| 12971| 0.9%  
> Vinod Koul| 12944| 0.9%  
> Marek Belisko| 11531| 0.8%  
> Al Cho| 10097| 0.7%  
  
These lists feature a combination of old and new names. Chris Wilson got to the top of the by-changesets list by virtue of his work with the Intel graphics drivers. Greg Kroah-Hartman worked on the USB and TTY subsystems, but the bulk of his changes were made to the staging tree, and the new brcm80211 driver in particular. Eric Dumazet has been active all over the networking layer, Johannes Berg has been busy with wireless networking, and Thomas Gleixner rewrote the generic interrupt handling code (among many other things). 

Looking at lines changed: Henry Ptasinski arrived at the top of the list through the addition of the brcm80211 driver which, as can be seen, is not a small piece of code. Vipin Mehta added the ath6kl driver to the staging tree, Luis Rodriguez worked on the mac80211 layer and the ath9k driver, and Mark Brown, as always, did a massive amount of work within the ALSA sound layer. It's interesting to note that five of the top ten in this column were mainly involved with the staging tree. One of the others (Krishna Gudipati) submitted a single "driver cleanup patch" which drew Linus's ire at the time - calling a nearly 40,000-line patch a "cleanup" seemed like a bit much. 

A minimum of 193 employers supported work on the 2.6.37 kernel; the top supporters were: 

> Most active 2.6.37 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1864| 16.6%  
> Red Hat| 1265| 11.3%  
> (Unknown)| 946| 8.4%  
> Intel| 746| 6.6%  
> Novell| 644| 5.7%  
> IBM| 447| 4.0%  
> Oracle| 272| 2.4%  
> Texas Instruments| 250| 2.2%  
> Nokia| 235| 2.1%  
> Renesas Technology| 210| 1.9%  
> Samsung| 205| 1.8%  
> Broadcom| 172| 1.5%  
> Societe Française de Radiotelephone| 166| 1.5%  
> AMD| 163| 1.5%  
> (Consultant)| 155| 1.4%  
> Pengutronix| 154| 1.4%  
> (Academia)| 146| 1.3%  
> Wolfson Micro| 129| 1.1%  
> Google| 128| 1.1%  
> Fujitsu| 127| 1.1%  
> | By lines changed  
> ---  
> Broadcom| 246749| 16.4%  
> (None)| 188830| 12.6%  
> Atheros| 165693| 11.0%  
> Novell| 127476| 8.5%  
> Wolfson Micro| 100196| 6.7%  
> Brocade| 64012| 4.3%  
> (Unknown)| 59325| 4.0%  
> Red Hat| 59000| 3.9%  
> Intel| 53893| 3.6%  
> Cypress Semiconductor| 48241| 3.2%  
> Vyatta| 30720| 2.0%  
> Texas Instruments| 22874| 1.5%  
> Samsung| 16771| 1.1%  
> IBM| 16567| 1.1%  
> Nokia| 14731| 1.0%  
> Renesas Technology| 14488| 1.0%  
> (Consultant)| 13592| 0.9%  
> Slimlogic Ltd| 13324| 0.9%  
> ST Ericsson| 13056| 0.9%  
> Chelsio| 10770| 0.7%  
  
Looking at the by-changesets column, the situation looks mostly like business as usual. Red Hat remains, by far, the largest corporate contributor to the kernel. On the lines-changed side, instead, Red Hat had to settle for eighth place behind companies which, for the most part, have contributed a lot of driver code. 

It has been some time since we looked at the reviewing and testing of code. In the 2.6.37 development cycle, the developers with the most Reviewed-by tags were: 

> Most active 2.6.37 reviewers  
> ---  
> Ingo Molnar| 86| 12.6%  
> Christoph Hellwig| 37| 5.4%  
> Mike Christie| 29| 4.3%  
> Michael Chan| 26| 3.8%  
> Josh Triplett| 22| 3.2%  
> Daniel Vetter| 21| 3.1%  
> Chuck Lever| 21| 3.1%  
> H. Peter Anvin| 20| 2.9%  
> Konrad Rzeszutek Wilk| 19| 2.8%  
> Alex Elder| 17| 2.5%  
> Luciano Coelho| 16| 2.3%  
> Suresh Jayaraman| 16| 2.3%  
> Swen Schillig| 16| 2.3%  
> Michal Marek| 15| 2.2%  
> Jeff Layton| 15| 2.2%  
> Sam Ravnborg| 15| 2.2%  
> Wu Fengguang| 12| 1.8%  
> Francisco Jerez| 12| 1.8%  
> KOSAKI Motohiro| 12| 1.8%  
> Christoph Lameter| 11| 1.6%  
  
As always, there is only so much that can be learned from these numbers; the bulk of all patch reviews do not lead to the addition of a Reviewed-by tag. On the other hand, there has been some real social pressure to credit users who test patches and report bugs: 

> Most credited 2.6.37 reporters and testers  
> ---  
> | `Reported-by` credits  
> ---  
> Stephen Rothwell| 24| 4.2%  
> Randy Dunlap| 21| 3.7%  
> Linus Torvalds| 19| 3.3%  
> Ingo Molnar| 15| 2.6%  
> Guennadi Liakhovetski| 10| 1.8%  
> Jonathan Cameron| 7| 1.2%  
> David Brownell| 7| 1.2%  
> Sitsofe Wheeler| 6| 1.1%  
> Andrew Morton| 6| 1.1%  
> Dan Rosenberg| 6| 1.1%  
> Jiri Slaby| 5| 0.9%  
> Ben Greear| 5| 0.9%  
> Eric Dumazet| 4| 0.7%  
> Daniel Vetter| 4| 0.7%  
> David S. Miller| 4| 0.7%  
> Robin Holt| 4| 0.7%  
> Andi Kleen| 4| 0.7%  
> Dan Carpenter| 4| 0.7%  
> Sachin Sant| 4| 0.7%  
> Dr. David Alan Gilbert| 4| 0.7%  
> | `Tested-by:` credits  
> ---  
> Wolfram Sang| 15| 4.1%  
> Luciano Coelho| 11| 3.0%  
> Kevin Hilman| 10| 2.7%  
> Jeff Pieper| 10| 2.7%  
> Will Deacon| 9| 2.5%  
> Caglar Akyuz| 8| 2.2%  
> Michael Williamson| 8| 2.2%  
> Emil Tantilov| 7| 1.9%  
> Randy Dunlap| 6| 1.6%  
> Stephen Ko| 6| 1.6%  
> Ben Greear| 6| 1.6%  
> Eric Benard| 5| 1.4%  
> Tuomas Katila| 5| 1.4%  
> Maxim Levitsky| 5| 1.4%  
> Ingo Molnar| 4| 1.1%  
> Juuso Oikarinen| 3| 0.8%  
> Jason Wessel| 3| 0.8%  
> Kuninori Morimoto| 3| 0.8%  
> Rabin Vincent| 3| 0.8%  
> Ben Gardiner| 3| 0.8%  
  
All told, there were 568 Reported-by and 366 Tested-by tag lines found in patches merged for 2.6.37. We are, it seems, slowly getting better at recognizing the people who are doing this crucially important work. 

In summary, the kernel development process continues to look healthy. We have a great deal of activity from an increasing number of developers, while, it seems, keeping a reasonable lid on the number of regressions introduced. Whether the high patch rate will continue into 2.6.38 remains to be seen; as of this writing, there are just under 5,000 changes in linux-next. Unless the subsystem maintainers put more work into linux-next in the near future, the next development cycle could be relatively slow.
