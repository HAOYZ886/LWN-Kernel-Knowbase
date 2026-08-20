---
title: Removing ext2 and/or ext3
url: https://lwn.net/Articles/426961/
date: "February 9, 2011"
category: "Filesystems-ext4"
author: "By Jonathan Corbet February 9, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
February 9, 2011

The ext4 filesystem has, at this point, moved far beyond its experimental phase. It is now available in almost all distributions and is used by default in many of them. Many users may soon be in danger of forgetting that the ext2 and ext3 filesystems even exist in the kernel. But those filesystems do exist, and they require ongoing resources to maintain. Keeping older, stable filesystems around makes sense when the newer code is stabilizing, but somebody is bound to ask, sooner or later, whether it is time to say "goodbye" to the older code. 

The question, as it turns out, came [sooner](<https://lwn.net/Articles/426962/>) \- February 3, to be exact - when Jan Kara suggested that removing ext2 and ext3 could be discussed at the upcoming storage, filesystems, and memory management summit. Jan asked: 

Of course it costs some effort to maintain them all in a reasonably good condition so once in a while someone comes and proposes we should drop one of ext2, ext3 or both. So I'd like to gather input what people think about this - should we ever drop ext2 / ext3 codebases? If yes, under what condition do we deem it is OK to drop it? 

One might protest that there will be existing filesystems in the ext3 (and even ext2) formats for the indefinite future. Removing support for those formats is clearly not something that can be done. But removing the ext2 and/or ext3 code is not the same as removing support: ext4 has been very carefully written to be able to work with the older formats without breaking compatibility. One can mount an ext3 filesystem using the ext4 code and make changes; it will still be possible to mount that filesystem with the ext3 code in the future. 

So it is possible to remove ext2 and ext3 without breaking existing users or preventing them from going back to older implementations. Beyond that, mounting an ext2/3 filesystem under ext4 allows the system to use a number of performance enhancing techniques - like delayed allocation - which do not exist in the older implementations. In other words, ext4 can replace ext2 and ext3, maintain compatibility, and make things faster at the same time. Given that, one might wonder why removing the older code even requires discussion. 

There appear to be a couple of reasons not to hurry into this change, both of which have to do with testing. As Eric Sandeen [noted](<https://lwn.net/Articles/426964/>), some of the more ext3-like options are not tested as heavily as the native modes of operation: 

ext4's more, um ... unique option combinations probably get next to no testing in the real world. So while we can say that noextent, nodelalloc is mostly like ext3, in practice, does that ever really get much testing? 

There is also concern that ext4, which is still seeing much more change than its predecessors, is more likely to introduce instabilities. That's a bit of a disturbing idea; there are enough production users of ext4 now that the introduction of serious bugs would not be pleasant. But, again, the backward-compatible operating modes of ext4 may not be as heavily tested as the native mode, so one might argue that operation with older filesystems is more likely to break regardless of how careful the developers are. 

So, clearly, any move to get rid of ext2 and ext3 would have to be preceded by the introduction of better testing for the less-exercised corners of ext4. The developers involved understand that clearly, so there is no need to be worried that the older code could be removed too quickly. 

Meanwhile, there are also concerns that the older code, which is not seeing much developer attention, could give birth to bugs of its own. As Jan [put it](<https://lwn.net/Articles/426965/>): 

The time I spend is enough to keep ext3 in a good shape I believe but I have a feeling that ext2 is slowly bitrotting. Sometime when I look at ext2 code I see stuff we simply do differently these days and that's just a step away from the code getting broken... It would not be too much work to clean things up and maintain but it's a work with no clear gain (if you do the thankless job of maintaining old code, you should at least have users who appreciate that ;) so naturally no one does it. 

Developers have also expressed concern that new filesystem authors might copy code from ext2, which, at this point, does not serve as a good example for how Linux filesystems should be written. 

The end result is that, once the testing concerns have been addressed, everybody involved might be made better off by the removal of ext2 and ext3. Users with older filesystems would get better performance and a code base which is seeing more active development and maintenance. Developers would be able to shed an older maintenance burden and focus their efforts on a single filesystem going forward. Thanks to the careful compatibility work which has been done over the years, it may be possible to safely make this move in the relatively near future.
