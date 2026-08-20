---
title: Some numbers and thoughts on the stable kernels
url: https://lwn.net/Articles/402512/
date: "August 27, 2010"
category: "Development model-Stable tree; Releases-Stable updates"
author: "By Jonathan Corbet August 27, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 27, 2010

Much attention goes toward mainline kernel releases, but relatively few users are actually running those kernels. Instead, they run kernels provided by their distributors, and those kernels, in turn, are based off the stable kernel series. The practice of releasing stable kernels has been going for well over five years now, so perhaps it's time to look back at how it has been going. 

Back in March 2005, the community was discussing ways of getting important fixes out to users of mainline releases. There was talk of maintaining a separate tree containing nothing but fixes; Linus, at the time, [thought that any such attempt was doomed to failure](<http://article.gmane.org/gmane.linux.kernel/283396>): 

I'll tell you what the problem is: I don't think you'll find anybody to do the parallel "only trivial patches" tree. They'll go crazy in a couple of weeks. Why? Because it's a _damn_ hard problem. Where do you draw the line? What's an acceptable patch? And if you get it wrong, people will complain _very_ loudly, since by now you've "promised" them a kernel that is better than the mainline. In other words: there's almost zero glory, there are no interesting problems, and there will absolutely be people who claim that you're a dick-head and worse, probably on a weekly basis. 

With such strong words of encouragement, somebody clearly had to step up to the job; that somebody turned out to be Greg Kroah-Hartman and Chris Wright. They released [2.6.11.1](<http://lwn.net/Articles/126324/>) on March 4, 2005 with all of three fixes. More than five years later, Greg (a.k.a. "[Og](<http://lwn.net/Articles/402327/>)") is still at it (Chris has not been active with stable updates for a while). During that time, the stable release history has looked like this: 

> Kernel | Updates| Changes  
> ---|---|---  
> Total| Per release  
> 2.6.11| 12| 79| 7  
> 2.6.12| 6| 53| 9  
> 2.6.13| 5| 44| 9  
> 2.6.14| 7| 96| 14  
> 2.6.15| 7| 110| 16  
>    
> 2.6.16| 62| 1053| 17  
> 2.6.17| 14| 191| 14  
> 2.6.18| 8| 240| 30  
> 2.6.19| 7| 189| 27  
> 2.6.20| 21| 447| 21  
>    
> 2.6.21| 7| 162| 23  
> 2.6.22| 19| 379| 20  
> 2.6.23| 16| 302| 19  
> 2.6.24| 7| 246| 35  
> 2.6.25| 20| 492| 25  
>    
> 2.6.26| 8| 321| 40  
> **2.6.27** | 53| 1553 | 29  
> 2.6.28| 10| 613| 61  
> 2.6.29| 6| 383| 64  
> 2.6.30| 10| 419| 42  
>    
> 2.6.31| 14| 826| 59  
> **2.6.32** | 21| 1793| 85  
> 2.6.33| 7| 883| 126  
> 2.6.34| 5| 599| 120  
> **2.6.35** | 4| 228| 57  
  
In the table above, the kernels appearing in **bold** are those which are still receiving stable updates as of this writing (though 2.6.27 is clearly reaching the end of the line). 

A couple of conclusions immediately jump out of the table above. The first is that the number of fixes going into stable updates has clearly increased over time. From this one might conclude that our kernel releases have steadily been getting buggier. That is hard to measure, but one should bear in mind that there is another important factor at work here: the kernel developers are simply directing more fixes toward the stable tree. Far more developers are looking at patches with stable updates in mind, and suggestions that a patch should be sent in that direction are quite common. So far fewer patches fall through the cracks than they did in the early days. 

There is another factor at work here as well. The initial definition of what constituted an appropriate stable tree patch was severely restrictive; if a bug did not cause a demonstrable oops or vulnerability, the fix was not considered for the stable tree. By the time we get to the [2.6.35.4 update](<http://lwn.net/Articles/402332/>), though, we see a wider variety of "fixes," including Acer rv620 laptop support, typo fixes, tracepoint improvements to make powertop work better, the [optimistic spinning mutex scalability work](<http://lwn.net/Articles/401911/>), a new emu10k1 sound driver module parameter, and oprofile support for a new Intel processor. These enhancements are, arguably, all things that stable kernel users would like to have. But they definitely go beyond the original charter for this tree. 

Your editor has also, recently, seen an occasional complaint about regressions finding their way into stable updates; given the volume of patches going into stable updates now, a regression every now and then should not be surprising. Regressions in the stable tree are a worrisome prospect; one can only hope that the problem does not get worse. 

Another noteworthy fact is that the number of stable updates for most kernels appears to be falling slowly; the five updates for the entire 2.6.34 update history is the lowest ever, matched only by the 2.6.13 series. Even then, 2.6.34 got one more update than had been originally planned as the result of a security issue. It should seem obvious that handling this kind of patch flow for as many as four kernels simultaneously will be a lot of work; Greg, who has a few other things on his plate as well, may be running a little short on time. 

Who is actually contributing patches to stable kernels? Your editor decided to do a bit of git data mining. Two kernels were chosen: 2.6.32, which is being maintained for an extended period as the result of its use in "enterprise" distributions, and 2.6.34, being the most recent kernel which has seen its final stable update. Here are the top contributors for both: 

> Most active stable contributors  
> ---  
> | 2.6.32  
> ---  
> Greg Kroah-Hartman| 36| 2.0%  
> Daniel T Chen| 32| 1.8%  
> Linus Torvalds| 23| 1.3%  
> Trond Myklebust| 23| 1.3%  
> Borislav Petkov| 23| 1.3%  
> Ben Hutchings| 21| 1.2%  
> David S. Miller| 20| 1.1%  
> Theodore Ts'o| 20| 1.1%  
> Tejun Heo| 20| 1.1%  
> Dmitry Monakhov| 20| 1.1%  
> Takashi Iwai| 18| 1.0%  
> Ian Campbell| 18| 1.0%  
> Jean Delvare| 17| 0.9%  
> Henrique de Moraes Holschuh| 17| 0.9%  
> Yan, Zheng| 17| 0.9%  
> Zhao Yakui| 17| 0.9%  
> Alan Stern| 17| 0.9%  
> Al Viro| 16| 0.9%  
> Alex Deucher| 15| 0.8%  
> Dan Carpenter| 15| 0.8%  
> | 2.6.34  
> ---  
> Alex Deucher| 14| 2.8%  
> Joerg Roedel| 14| 2.8%  
> Tejun Heo| 10| 2.0%  
> Daniel T Chen| 9| 1.8%  
> Neil Brown| 8| 1.6%  
> Rafael J. Wysocki| 8| 1.6%  
> Linus Torvalds| 7| 1.4%  
> Greg Kroah-Hartman| 7| 1.4%  
> Alan Stern| 7| 1.4%  
> Jesse Barnes| 7| 1.4%  
> Trond Myklebust| 7| 1.4%  
> Ben Hutchings| 7| 1.4%  
> Tilman Schmidt| 7| 1.4%  
> Avi Kivity| 7| 1.4%  
> Sarah Sharp| 7| 1.4%  
> Ian Campbell| 6| 1.2%  
> Johannes Berg| 6| 1.2%  
> Jean Delvare| 6| 1.2%  
> Johan Hovold| 6| 1.2%  
> Will Deacon| 5| 1.0%  
  
Some names on this list will be familiar. Linus never shows up on the list of top mainline contributors anymore, but he does generate a fair number of stable fixes. Other names are seen less often in the kernel context: Daniel Chen, for example, is an Ubuntu community contributor; his contributions are mostly in the welcome area of making audio devices actually work. Some of the people are in the list above because they introduced the bugs that their patches fix - appearing in that role is not necessarily an honor. But - admittedly without having done any sort of rigorous study - your editor suspects that most of the people listed above are fixing bugs introduced by others. They are performing an important and underappreciated service, turning mainline releases into kernels that the rest of the world actually wants to run. 

We can also look at who is supporting this work: 

> Most active stable contributors by employer  
> ---  
> | 2.6.32  
> ---  
> (None)| 275| 15.3%  
> Red Hat| 267| 14.9%  
> Intel| 194| 10.8%  
> (Unknown)| 175| 9.8%  
> Novell| 166| 9.3%  
> IBM| 95| 5.3%  
> AMD| 60| 3.3%  
> Oracle| 40| 2.2%  
> Fujitsu| 33| 1.8%  
> Atheros| 30| 1.7%  
> Parallels| 29| 1.6%  
> Citrix| 27| 1.5%  
> (Academia)| 26| 1.5%  
> Linux Foundation| 24| 1.3%  
> NetApp| 23| 1.3%  
> Google| 23| 1.3%  
> (Consultant)| 20| 1.1%  
> NEC| 18| 1.0%  
> Canonical| 15| 0.8%  
> Nokia| 14| 0.8%  
> | 2.6.34  
> ---  
> (None)| 95| 18.7%  
> Red Hat| 61| 12.0%  
> (Unknown)| 58| 11.4%  
> Novell| 45| 8.9%  
> Intel| 43| 8.5%  
> AMD| 35| 6.9%  
> IBM| 17| 3.3%  
> (Academia)| 16| 3.1%  
> MontaVista| 9| 1.8%  
> Fujitsu| 9| 1.8%  
> ARM| 8| 1.6%  
> Citrix| 8| 1.6%  
> NetApp| 7| 1.4%  
> Oracle| 7| 1.4%  
> (Consultant)| 7| 1.4%  
> Linux Foundation| 7| 1.4%  
> Google| 6| 1.2%  
> Nokia| 6| 1.2%  
> NTT| 5| 1.0%  
> VMWare| 5| 1.0%  
  
These numbers quite closely match those for mainline kernel contributions, especially at the upper end. Fixing bugs is said to be boring and unglamorous work, but volunteers are still our leading source of fixes. 

We did without a stable tree for the first ten 2.6.x releases, though, at this point, it's hard to imagine just how. In an ideal world, a mainline kernel release would not happen until there were no bugs left; the history of (among others) the 2.3 and 2.5 kernel development cycles shows that this approach does not work in the real world. There comes a point where the community has to create a stable release and go on to the next cycle; the stable tree allows that fork to happen without ending the flow of fixes into the released kernel. 

The tables above suggest that the stable kernel process is working well, with large numbers of fixes being directed into stable updates and with participation from across the community. There may come a point, though, where that community needs to revisit the standards for patches going into stable updates. At some point, it may also become clear that the job of maintaining these kernels is too big for one person to manage. For now, though, the stable tree is clearly doing what it is intended to do; Greg deserves a lot of credit for making it work so well for so long.
