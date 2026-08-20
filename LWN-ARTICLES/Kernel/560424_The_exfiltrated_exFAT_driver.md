---
title: The exfiltrated exFAT driver
url: https://lwn.net/Articles/560424/
date: "July 24, 2013"
category: "Copyright issues; Filesystems-exFAT"
author: "By Jonathan Corbet July 24, 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 24, 2013

The [exFAT filesystem](<https://en.wikipedia.org/wiki/ExFAT>) is a Microsoft product, designed for flash media. It lacks support in the Linux kernel; as a proprietary, heavily patented filesystem, it is not the sort of thing one would expect to see free support for. Still, when the [exfat-nofuse repository](<https://github.com/rxrz/exfat-nofuse>) showed up on GitHub, some dared to hope that Linux would gain exFAT support after all. Instead, what we appear to have gained is an ugly licensing mess and code that is best avoided. 

From what can be determined by looking at the repository, the code appears to work. It was originally written by Samsung, it seems, and was shipped with one or more Android devices. The problem is that, as far as anybody can tell, Samsung never intended to distribute this code under the GPL. Instead, a GitHub user who goes by "rxrz" somehow came by a copy of the code, removed the original proprietary licensing headers, and inserted a GPL license declaration into the code. The code _claimed_ to have a GPL license, but the copyright owner never released the code under that license. 

On July 9, another GitHub user [filed a bug](<https://github.com/rxrz/exfat-nofuse/issues/5>) noting that the license declaration was incorrect and suggesting a removal of the repository. The entity known as rxrz was not impressed, though, saying: 

It's a leaked code of a proprietary exfat driver, written by Samsung, Inc. It works, you can use it. What else do you want, a signed paper from your parents on whether you can or can not use it? I'm a programmer, not a lawyer. You got the code, now decide what to do with it, it's up to you. 

The code has since been edited to remove the GPL declaration and restore the proprietary license, but it remains available on GitHub and rxrz evidently feels that nothing wrong was done by posting it there. It also appears that GitHub has no interest in pulling down the repository in the absence of an explicit takedown notice from Samsung, so this "leaked" driver may remain available for some time. 

This whole episode seems like a fairly straightforward case of somebody trying to liberate proprietary code by any means available. There are some interesting questions raised by all of this, though. The first of those is: what if somebody had tried to merge this code into the mainline kernel? The immediate answer is that they would have been chased off the list once developers actually had a look at the code, which, to put it gently, does not much resemble Linux kernel code. In the absence of this obvious barrier, one can hope that our normal review mechanisms would have kept this code from being merged until the developer was able to provide a satisfactory explanation of where it came from. 

But it is not clear that all of our code is reviewed to that level, so it is hard to be sure. An exFAT implementation is likely to attract enough attention to ensure that the right questions are asked. Had the code in question been a driver for a relatively obscure piece of hardware, instead, it might not have been looked at very closely. 

Then, one might ask: why is Samsung shipping this as a proprietary module in the first place? After all, Samsung appears to have figured out how Linux kernel development works and has made a solid place for itself as one of the largest contributors to the kernel. One can only guess at the answer, but it likely has to do with claims that Microsoft makes over the exFAT format. Microsoft has shown itself to be willing to assert patents on filesystem formats, so taking some care with an implementation of a new Microsoft filesystem format would seem like an exercise in basic prudence. Whether this exercise led to ignoring the GPL in an imprudent manner is the subject of another debate entirely. 

Similarly, some prudence would be advisable for anybody thinking to use the code as a reverse-engineering tool for a new exFAT implementation. It is hard to reverse-engineer one's way around patent problems. exFAT may well be a format that is best left alone. 

Finally, for those who have been in this community for a long time, the attitude revealed by a number of participants in the GitHub issue thread may be surprising. Licensing, GPL or otherwise, appears not to matter to many of these people. All that matters is that the code can be downloaded and that it works. This attitude can be found elsewhere on GitHub; indeed, many have complained that GitHub itself seems to be indifferent at best to the licensing of the code it distributes. 

Perhaps we are heading into some sort of post-copyright era where licensing truly no longer matters. But it would not be surprising if those who are interested in copyright resist that future for a while yet. We are not just talking about the entertainment industry here; the simple fact of the matter is that anybody who values the provisions of the GPL is indeed interested in copyright. It is hard to demand respect for the GPL while refusing to respect the terms of other licenses. 

Among other things, that means that the kernel community must continue to be careful not to incorporate code that has not been contributed under a suitable license. So code that shows up on the net must be looked at carefully, no matter how useful it appears to be. In this case, there was no danger that the exFAT code would ever be merged; nobody even suggested that it should be. But there will be other modules of dubious provenance in the future, some of which may seem more legitimate at first glance. Even then, though, our processes should be good enough to find the problems and avoid a merger that we will later regret. Hopefully. 

(Thanks to Armijn Hemel for the heads-up).
