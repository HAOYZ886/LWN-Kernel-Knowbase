---
title: "Who's writing 2.6.21 and related issues"
url: https://lwn.net/Articles/224760/
date: "March 7, 2007"
category: "Development model; Development model-Contributor statistics; Releases-2.6.21"
author: 
---

Our article [Who wrote 2.6.20?](<http://lwn.net/Articles/222773/>), which appeared two weeks ago, generated a strong response. There is, it seems, a lot of interest in where this code is coming from, but nobody had gotten around to doing the crunching to figure it out. That article calls for a followup in a few ways. 

First, those who saw the article early on may want to take another look, as some of the tables have been changed. There was only one serious mistake to fix - one developer's affiliation was incorrectly guessed by the code - but further information has also helped to shrink the "unknown" column somewhat. The original tables can be found from the article (for whatever historical reasons may exist), but the tables in the article itself are the current ones. 

The 2.6.21 cycle has moved far enough along as of this writing (the 2.6.21-rc3 prepatch is due any time) that it's worth taking a look at the statistics for the just over 4,000 changesets which have been merged. There are some familiar names here, but some new ones as well. The reflect the different nature of this development cycle, 2.6.21 will have fewer changes in the virtualization area, for example, but it has some significant core changes (like the [clockevents and dynamic tick](<http://lwn.net/Articles/223185/>) work). A somewhat different set of developers had work ready to merge this time around, and the results show that. 

Anyway, the developers with the most work merged this time around are: 

> Most active 2.6.21 developers  
> ---  
> By changesets|   | By lines changed  
> Eric W. Biederman| 104| 2.5% |   | Adrian Bunk| 24097| 6.1%  
> Ralf Baechle| 77| 1.9% |   | Divy Le Ray| 18255| 4.6%  
> Adrian Bunk| 71| 1.7% |   | Ben Dooks| 17510| 4.4%  
> Bob Moore| 66| 1.6% |   | Andrew Victor| 13877| 3.5%  
> Andrew Morton| 54| 1.3% |   | Ralf Baechle| 9905| 2.5%  
> Takashi Iwai| 54| 1.3% |   | YOSHIFUJI Hideaki| 9505| 2.4%  
> Robert P. J. Day| 53| 1.3% |   | Steve Wise| 9418| 2.4%  
> Jeff Dike| 52| 1.3% |   | Jeff Garzik| 7014| 1.8%  
> Jiri Slaby| 51| 1.2% |   | Vitaly Bordug| 6387| 1.6%  
> Ben Dooks| 50| 1.2% |   | Thomas Gleixner| 6078| 1.5%  
> Tejun Heo| 48| 1.2% |   | Bob Moore| 6055| 1.5%  
> Al Viro| 48| 1.2% |   | Ishizaki Kou| 5912| 1.5%  
> David Brownell| 47| 1.1% |   | Richard Purdie| 5909| 1.5%  
> YOSHIFUJI Hideaki| 44| 1.1% |   | Liam Girdwood| 5773| 1.5%  
> Mike Isely| 43| 1.1% |   | Frank Mandarino| 5284| 1.3%  
> Thomas Gleixner| 38| 0.9% |   | Jay Cliburn| 5182| 1.3%  
> Randy Dunlap| 38| 0.9% |   | Tejun Heo| 5120| 1.3%  
> Stephen Hemminger| 36| 0.9% |   | Kumar Gala| 5044| 1.3%  
> Alan Cox| 35| 0.9% |   | Martin Schwidefsky| 4729| 1.2%  
> Michael Krufky| 32| 0.8% |   | Olof Johansson| 4659| 1.2%  
  
On the side of removing code, the list of names remains about the same: 

> Developers with the most lines removed  
> ---  
> Adrian Bunk| 23720| 12.8%  
> Jeff Garzik| 6808| 3.7%  
> Paul Mundt| 2442| 1.3%  
> Bob Moore| 1526| 0.8%  
> Len Brown| 1244| 0.7%  
> Alexey Starikovskiy| 987| 0.5%  
> Jiri Slaby| 954| 0.5%  
> Kenji Kaneshige| 661| 0.4%  
> Eric Sandeen| 609| 0.3%  
> Tim Schmielau| 547| 0.3%  
  
Adrian Bunk continues to remove code from the kernel at an amazing rate. Also about the same is the table of signoffs: 

> Developers with the most signoffs (total 8614)  
> ---  
> Andrew Morton| 1000| 11.6%  
> Linus Torvalds| 865| 10.0%  
> Jeff Garzik| 346| 4.0%  
> Jaroslav Kysela| 224| 2.6%  
> Greg Kroah-Hartman| 224| 2.6%  
> David Miller| 208| 2.4%  
> Mauro Carvalho Chehab| 206| 2.4%  
> Len Brown| 202| 2.3%  
> Takashi Iwai| 187| 2.2%  
> Ralf Baechle| 156| 1.8%  
> Russell King| 153| 1.8%  
> Paul Mackerras| 151| 1.8%  
> James Bottomley| 114| 1.3%  
> Eric W. Biederman| 105| 1.2%  
> Adrian Bunk| 99| 1.1%  
> Andi Kleen| 94| 1.1%  
> Alexey Starikovskiy| 82| 1.0%  
> Kyle McMartin| 79| 0.9%  
> David Brownell| 78| 0.9%  
> Ingo Molnar| 68| 0.8%  
  
The list of developers contributing code to a given kernel release can change over time, but the people through whom those patches pass - the subsystem maintainers - remain about the same. These developers form the infrastructure which does the work of getting reviewed code into the mainline kernel. 

Here's the by-employer tables for 2.6.21-rc: 

> Top contributors by employer  
> ---  
> By changesets |   | By lines changed  
> (Unknown)| 1108| 27.1% |   | (Unknown)| 85436| 21.5%  
> (None)| 380| 9.3% |   | (None)| 52312| 13.2%  
> Red Hat| 304| 7.4% |   | IBM| 28186| 7.1%  
> Intel| 280| 6.8% |   | Intel| 20778| 5.2%  
> IBM| 259| 6.3% |   | Red Hat| 19007| 4.8%  
> Novell| 258| 6.3% |   | Novell| 18702| 4.7%  
> Linux Foundation| 159| 3.9% |   | Chelsio| 18361| 4.6%  
> Linux Networx| 104| 2.5% |   | Simtec| 17545| 4.4%  
> (Consultant)| 100| 2.4% |   | SANPeople| 13949| 3.5%  
> Oracle| 89| 2.2% |   | MIPS Technologies| 12646| 3.2%  
> MIPS Technologies| 77| 1.9% |   | Open Grid Computing| 9442| 2.4%  
> Google| 61| 1.5% |   | MontaVista| 8861| 2.2%  
> MontaVista| 55| 1.3% |   | Toshiba| 7462| 1.9%  
> SGI| 54| 1.3% |   | Wolfson Microelectronics| 7379| 1.9%  
> Simtec| 50| 1.2% |   | Sony| 7061| 1.8%  
> Nokia| 41| 1.0% |   | Freescale| 6993| 1.8%  
> TimeSys| 38| 0.9% |   | TimeSys| 6184| 1.6%  
> Sony| 36| 0.9% |   | Endrelia| 5421| 1.4%  
> HP| 35| 0.9% |   | Nokia| 4790| 1.2%  
> Toshiba| 34| 0.8% |   | Renesas Technology| 4740| 1.2%  
  
Many of the names are the same, but Red Hat does not dominate to quite the same extent as in 2.6.20. The percentage of patches contributed by developers known to be working on their own time has increased slightly. 

Finally, some commenters on the original article requested the release of the code used to generate the numbers. Your editor has some qualms about doing so. The biggest among them is _not_ that the code is an embarrassing hack with, presumably, at least one bug still in it. Neither is it the fact that the code could be seen as a competitive tool for LWN; frankly, there's nothing _that_ complicated there. 

The biggest worry is related to the attention these numbers drew, and the fact that a couple of developers have mailed in to note that they have received job offers as a result of appearing in the LWN lists. In addition, a few employers have contacted us to be sure that their "account" is credited with the work of all of their employees. The numbers your editor has generated are approximations, but some people clearly see them as being important. 

The editors at LWN have an interest in covering the free software community while minimizing the changes that such coverage might cause - most of the time, at least. It seems plausible that, if the "top 20 contributors list" is seen as a desirable place to appear - with positive career benefits - developers might change their behavior as a result. It would be a shame to start seeing kernel patches aimed mainly at increasing a developer's count of lines changed. Such patches, one assumes, would not fare well in the review process, but it would be better if the situation did not come up at all. 

The issue of the mapping between developers and their employers is also worth some consideration. Some of that information was obtained directly from the developers with a promise not to disclose it further; that promise must be kept. Beyond that, developers tend to change employers over time, and the code is not currently smart enough to deal with that. This shortcoming is not a problem when looking at a single release cycle, but it clearly would be an issue for multi-year analysis. The code could be improved, but it's not at all clear that the maintenance and distribution of a database of kernel developers' work histories is something LWN wants to get into. There are serious privacy issues to consider. 

Despite these worries, the code is being released. In the end, it's not as if somebody else would have all that much trouble reproducing it. Some of the employer information has been taken out in response to the concerns outlined above, though. A tarball of the initial release can be found [here](</images/gitdm/>); your editor is looking forward to the flood of patches which will improve the system.
