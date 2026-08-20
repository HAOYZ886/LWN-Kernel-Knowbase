---
title: Long discussions about long names
url: https://lwn.net/Articles/331615/
date: "May 4, 2009"
category: Legal issues
author: "By Jonathan Corbet May 4, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 4, 2009

When Microsoft filed its lawsuit against TomTom, it named two patents which cover the VFAT filesystem. That, naturally, led to a renewed push to either (1) get those patents invalidated, or (2) move away from VFAT altogether. But some participants have advocated a third approach: find a way to work around the patents which retains most of the VFAT filesystem functionality while, with luck, avoiding any potential infringement of the claims of the patent. But, as a recently-posted [patch](<http://lwn.net/Articles/331322/>) and the ensuing discussion show, workarounds are not a straightforward solution even after the lawyers have been satisfied. 

The patch (written by Andrew Tridgell, but posted by Dave Kleikamp), comes with this changelog: 

Add CONFIG_VFAT_NO_CREATE_WITH_LONGNAMES option 

When this option is enabled the VFAT filesystem will refuse to create new files with long names. Accessing existing files with long names will continue to work. 

Note that the changelog gives no clue as to _why_ one might want this particular configuration option. What it probably comes down to is this: all of the claims in the VFAT patent refer to the _creation_ of long file names. Reading filesystems with such names is not addressed by the patent. So the apparent thinking is that, even if the named patents really read on the Linux VFAT implementation, they will not read on a version which cannot create files with long names. 

It looks like a reasonable hack. Interoperability with all existing VFAT filesystems is retained, as long as one does not need to create files with long names on the Linux side. But systems which run kernels with this option enabled have a much lower probability of being found to infringe on the VFAT patents. It could, maybe, be an optimal solution. 

That said, the patch has been poorly received in the kernel development community. One of the reasons for this chilly reception, certainly, is general hostility to the software patent system and an associated lack of willingness to capitulate to it. Add in a generous helping of contempt for the VFAT patents - and their owner - in particular, and it is not surprising that some developers would rather not entertain "solutions" to this problem. 

The bigger issue, though, is that the patch does not describe the real problem that it is trying to solve. There has been a lot of fairly weaselly discussion from IBM developers on the lists, but none of them are willing to just come out and say what is going on. The closest, perhaps, is [this message](<https://lwn.net/Articles/331626/>) from Tridge: 

However, if you are willing to concede that there are good non-technical arguments for wanting to "get the VFAT out" then choosing the best way to achieve that is most definitely a technical decision, and that is what we can discuss here. 

Unfortunately I am unable to discuss any of the non-technical reasons for why "get the VFAT out" might be a good idea in the first place. That is damn frustrating, but it is just how things are. 

All of this talk creates a certain feeling of patches being sent out to the list from some smoke-filled room deep within IBM headquarters. But, more importantly, the lack of information makes it impossible for the development community to determine whether the patch works. To make that decision, developers need to know what problem is being solved, and how the proposed solution makes the problem go away. But they don't have that information; instead, they simply have a patch which makes it possible to remove some functionality from the kernel. 

The subtext of the conversation is that some lawyers at IBM have, presumably, determined that a potential problem exists. That problem could be as simple as "this feature may attract infringement suits," independently of whether the patents are valid or whether Linux infringes on them. For any number of Linux users, the simple fact that the probability of being sued might go up is enough to inspire a search for alternatives. Also, presumably, these same lawyers have concluded that this particular workaround can resolve these worries. So now they believe it should be a part of the Linux kernel. 

But if the lawyers have really come to these conclusions, they are not saying so in any public forum. So the kernel developers are left wondering what is really going on. Are there really lawyers involved, or is this patch the work of a couple of programmers who have tried to create a solution (to a problem perceived by them) on their own? Why can't a company like TomTom just patch out the long-name functionality on their own if they are truly worried about it? Might the inclusion of this patch open the kernel up to other potential legal difficulties that we don't know about? 

Tridge's [suggestion](<https://lwn.net/Articles/331639/>) is that a prominent kernel developer needs to have a conversation with a lawyer before making the decision on this patch. That approach might lead to a correct outcome, but it will still leave most of the community in the dark and unhappy about it. 

It would appear that a better way is required. Currently, it is difficult for developers to determine whether a patent really applies to an algorithm in the kernel or not. If they conclude that there is a patent problem, these same developers are poorly placed to figure out what a minimal workaround might be. We need some help in this area. This particular problem is likely to come up again in other contexts; if we can put some sort of process in place for addressing legal issues, life will be easier in the future. 

IBM is said to have extensive documentation on the process of working around patents; for some strange reason, this information has never been released to the public. Unfortunately, determinations by lawyers are also unlikely to be released to the public, for any number of reasons. But developers need all of this information to respond properly to legal problems. There may be no alternative to some sort of process where a limited group of developers is given access to information under non-disclosure agreements. Such processes are distasteful, but they also are fairly common; many device drivers are created under non-disclosure agreements. 

The Linux Foundation currently has an NDA program intended to connect developers with hardware documentation. Perhaps a similar program (under the auspices of the Linux Foundation, or of another group like the Software Freedom Law Center or the Open Invention Network) could be created for access to legal information. As it is, we have a situation where some developers are talking to their employers' lawyers and nobody else has any real idea of what is going on. That will lead to slow, loud, and contentious attempts to solve legal problems. Given that we're almost certain to have more of these problems in the future, we might want to put some thought into finding a better way.
