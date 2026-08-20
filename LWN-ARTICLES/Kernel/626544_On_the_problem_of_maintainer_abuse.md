---
title: On the problem of maintainer abuse
url: https://lwn.net/Articles/626544/
date: "December 17, 2014"
category: "Development model-Code review"
author: "By Jonathan Corbet December 17, 2014"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 17, 2014

As can be seen in the [LWN kernel patch tracker](<https://lwn.net/Kernel/Patches>) (or in the patches section of the weekly Kernel Page), there have been a lot of significant patch sets posted over the course of the last week or so. The pace of kernel development continues to increase, so there is always a lot of new code out there in need of review. There's just one little potential problem: as of this writing, the 3.19 merge window is open, and many subsystem maintainers are busy getting the current set of changes into the mainline and dealing with any resulting fallout. They are unlikely to have much spare time for patch review. 

Merge-window patch postings are not uncommon; most maintainers either just defer looking at them or ignore them altogether. This time around, though, Thomas Gleixner [vented his frustration](<https://lwn.net/Articles/626545/>) on the linux-kernel list: 

Nothing of this is 3.19 material so posting it right now is just useless. I'm not going to look at it and I'm not going to look at it next week either. This whole featuritis driven 'post crap as fast as you can' thing has to stop, really. 

Posting patches during the merge window, he said, constitutes "maintainer abuse." 

Some developers agreed with these sentiments, and Kevin Cernekee promptly posted (during the merge window, of course) [a patch series titled "Stop maintainer abuse"](<https://lwn.net/Articles/626546/>) trying to codify a rule that patches should not be posted during merge windows. Patches meant for the next merge window should, by these rules, be posted prior to the preceding -rc5 release; patches posted after -rc5 comes out will end up coming out one release later. And no patches at all, other than urgent fixes, should be posted while the merge window is open. 

There is a potential problem, though, in that not all subsystem maintainers work the same way. Christoph Hellwig [disagreed](<https://lwn.net/Articles/626547/>) with the rules, saying: 

Merge window isn't really special, and patches can easily be reviewed and queued up for the next merge window in that time. If it said you shouldn't expect replies and not _resend_ during the merge window that seems like a much saner policy. 

Linus [responded to the posting guidelines](<https://lwn.net/Articles/626548/>) by noting that they don't apply equally to all maintainers: 

[F]or fairly simple subsystems in particular, some maintainers basically have their pull requests for the merge window open *before* the merge window even starts, and for them, the merge window itself isn't actually all that busy, it's often the week before that is the busy one. So the exact timing can vary by maintainership, and while I think the above is a reasonable example, it should perhaps be documented as such. 

Alan Cox [worried](<https://lwn.net/Articles/626549/>) that trying to put a lid on patch postings is always doomed to failure: 

Every time anyone has tried to deal with Linux scaling problems by throttling the rate it has failed, from the near forking of it when Linus couldn't cope onwards. Today we are already seeing the same occurring with all the vendor trees, and shared downstream trees with a rapidly growing amount of stuff that simply isn't upstream because upstream can't keep up with actual product timescales any more. 

His suggestion was to, instead, try to streamline the process a bit, mostly by improving the [patchwork](<https://patchwork.kernel.org/>) system to automate (or at least assist) many common maintainer duties. He finished by proposing that: ""It could then be integrated into git (if only so we can have a 'git lost' command to block annoying sources)."" 

In the end, it is hard to see this problem being solved by either more rules or better tools. The creation of kernel patches continues at an increasing pace; the kernel community has to keep up with the flow somehow or suffer in the long run. In many cases, what may really be needed is more maintainers; some subsystems are now maintained by groups, but most of them are still managed by a single developer. Spreading the load would allow some maintainers to work on merge window issues while others keep track of the patch flow. 

Such a change would require maintainers to allow others into their often fiercely guarded domains, though; the groups would also have to put time into developing a workflow that would work for them. It is not a simple or immediate solution, and it still will not address ills like developers who repost lengthy patch sets multiple times in one day. So, it seems, maintainers will still just have to get grumpy occasionally when developers push the boundaries too hard.
