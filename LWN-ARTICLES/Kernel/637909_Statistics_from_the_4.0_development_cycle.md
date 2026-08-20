---
title: Statistics from the 4.0 development cycle
url: https://lwn.net/Articles/637909/
date: "April 1, 2015"
category: "Releases-4.0"
author: "By Jonathan Corbet April 1, 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 1, 2015

The 4.0 kernel development cycle is heading toward its close; the final release can be expected on, mostly likely, April 12 (or possibly April 19 if a late regression turns up). This release will be notable for the new major version number, of course, even though the shift to 4.x does not mean anything more than "the minor numbers were getting too big." It may also be notable for being one of the slower development cycles in the last couple of years, though one should bear in mind that "slow" is used in a relative sense here. 

To be specific, as of this writing, the 4.0 cycle has seen the addition of just over 10,000 non-merge changesets to the mainline repository. 1,403 developers have contributed to this release cycle so far; they have added 403,000 lines of code and removed 222,000 for a net growth of 181,000 lines. The list of the most active developers looks a bit different than it has in the last few cycles: 

> Most active 4.0 developers  
> ---  
> | By changesets  
> ---  
> Lars-Peter Clausen| 179| 1.8%  
> Takashi Iwai| 172| 1.7%  
> H Hartley Sweeten| 153| 1.5%  
> Rickard Strandqvist| 111| 1.1%  
> Antti Palosaari| 95| 0.9%  
> Thierry Reding| 94| 0.9%  
> Geert Uytterhoeven| 88| 0.9%  
> Ian Abbott| 88| 0.9%  
> Maxime Ripard| 85| 0.8%  
> Michael S. Tsirkin| 82| 0.8%  
> Marcel Holtmann| 79| 0.8%  
> Ben Skeggs| 77| 0.8%  
> Arnd Bergmann| 76| 0.8%  
> Laurent Pinchart| 75| 0.7%  
> Rasmus Villemoes| 75| 0.7%  
> Al Viro| 71| 0.7%  
> Trond Myklebust| 71| 0.7%  
> Andy Shevchenko| 66| 0.7%  
> Krzysztof Kozlowski| 64| 0.6%  
> Christophe Ricard| 62| 0.6%  
> | By changed lines  
> ---  
> Ben Skeggs| 23587| 4.8%  
> Hans Verkuil| 16433| 3.3%  
> Thomas Petazzoni| 10642| 2.1%  
> Tero Kristo| 9341| 1.9%  
> Hariprasad Shenai| 8810| 1.8%  
> Michal Kazior| 7878| 1.6%  
> H Hartley Sweeten| 6925| 1.4%  
> Laurent Pinchart| 6803| 1.4%  
> Dudley Du| 5399| 1.1%  
> Takashi Iwai| 5137| 1.0%  
> Antti Palosaari| 4913| 1.0%  
> Boris Brezillon| 4666| 0.9%  
> Christoph Hellwig| 4365| 0.9%  
> Arnd Bergmann| 3974| 0.8%  
> Rusty Russell| 3963| 0.8%  
> Tony Lindgren| 3960| 0.8%  
> Rickard Strandqvist| 3921| 0.8%  
> Magnus Damm| 3771| 0.8%  
> Andrzej Pietrasiewicz| 3697| 0.7%  
> Maxime Ripard| 3664| 0.7%  
  
For once, Hartley Sweeten's work on the Comedi drivers did not put him at the top of the "by changesets" list; that place was taken by Lars-Peter Clausen, who worked mostly in the audio and media driver trees. Takashi Iwai's work remains entirely within the audio subsystem tree. Below Hartley, Richard Strandqvist cleaned up a lot of dead code throughout the tree, while Antti Palosaari did a lot of work in the media driver subsystem. 

In the "lines changed" column, Ben Skeggs carried out a massive renaming of symbols in the Nouveau driver; "nouveau" became "nvkm" in the parts of the code that make up the direct-rendering kernel module. Hans Verkuil continues to do work throughout the media subsystem; the bulk of his changed lines, though, took the form of removing some old, unloved drivers. Thomas Petazzoni added a number of framebuffer drivers to the staging tree, Tero Kristo cleaned up and enhanced the TI OMAP clock subsystem, and Hariprasad Shenai did a bunch of work on the cxgb4 network/InfiniBand drivers. 

For some years, work on the staging tree has tended to dominate these two lists, but that is not the case for 4.0. Indeed, this is one of the slowest development cycles for the staging tree in general, as can be seen in the plot below: 

> ![\[Staging patches plot\]](https://static.lwn.net/images/2015/staging-patches.png)

The slow traffic in the staging tree explains much of the relative slowness of the 4.0 development cycle in general. 

There are 197 employers who are known to have supported development of the 4.0 kernel. The most active of these were: 

> Most active 4.0 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1164| 11.6%  
> (None)| 864| 8.6%  
> (Unknown)| 712| 7.1%  
> Red Hat| 703| 7.0%  
> SUSE| 463| 4.6%  
> Linaro| 401| 4.0%  
> Samsung| 361| 3.6%  
> (Consultant)| 336| 3.3%  
> Free Electrons| 251| 2.5%  
> IBM| 226| 2.2%  
> Renesas Electronics| 188| 1.9%  
> Freescale| 165| 1.6%  
> Google| 154| 1.5%  
> Vision Engraving Systems| 153| 1.5%  
> Primary Data| 149| 1.5%  
> AMD| 142| 1.4%  
> Texas Instruments| 139| 1.4%  
> Oracle| 137| 1.4%  
> Qualcomm| 120| 1.2%  
> ARM| 119| 1.2%  
> | By lines changed  
> ---  
> Intel| 49062| 9.9%  
> Red Hat| 46588| 9.4%  
> (None)| 34171| 6.9%  
> Samsung| 23835| 4.8%  
> Free Electrons| 22943| 4.6%  
> Texas Instruments| 22395| 4.5%  
> (Unknown)| 22369| 4.5%  
> Cisco| 19438| 3.9%  
> Linaro| 17553| 3.5%  
> SUSE| 11764| 2.4%  
> Renesas Electronics| 10796| 2.2%  
> (Consultant)| 10719| 2.2%  
> Chelsio| 10439| 2.1%  
> IBM| 10355| 2.1%  
> Code Aurora Forum| 9697| 2.0%  
> Tieto| 8115| 1.6%  
> ARM| 7316| 1.5%  
> Vision Engraving Systems| 6925| 1.4%  
> Qualcomm| 6850| 1.4%  
> AMD| 6781| 1.4%  
  
In the [3.19 development statistics article](<https://lwn.net/Articles/631509/>), we noted that developers with no affiliation (volunteers) were holding steady at about 11% of the total changeset contribution. The accompanying suggestion that the decline in volunteer developers could be ending may have been premature, though; in 4.0, only 8.6% of the (relatively low) changeset count came from volunteers. That is the lowest point since 3.10 and the second-lowest ever. 

Otherwise, there is not much that jumps out from the above table; corporate support for kernel development doesn't change a whole lot from one development cycle to the next. 

As of this writing, there are about 6,500 non-merge changesets in the linux-next tree, putting linux-next in a place similar to where it was at this point in the 3.19 development cycle. That suggests that, unless things change, 4.1 will be another relatively slow cycle — though, it bears repeating, the incorporation of over 10,000 changes in a development cycle lasting less than three months is not all that slow. Even if we are not setting records at the moment, it seems clear that the kernel development community remains strong and active.
