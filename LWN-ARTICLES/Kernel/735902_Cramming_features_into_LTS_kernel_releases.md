---
title: Cramming features into LTS kernel releases
url: https://lwn.net/Articles/735902/
date: "October 10, 2017"
category: "Development model-Stable tree"
author: "By Jonathan Corbet October 10, 2017"
---

By **Jonathan Corbet**  
October 10, 2017

While the 4.14 development cycle has not been the busiest ever (12,500 changesets merged as of this writing, slightly more than 4.13 at this stage of the cycle), it has been seen as a rougher experience than its predecessors. There are all kinds of reasons why one cycle might be smoother than another, but it is not unreasonable to wonder whether the fact that 4.14 is a long-term support (LTS) release has affected how this cycle has gone. Indeed, when he released [4.14-rc3](<https://lwn.net/Articles/735217/>), Linus Torvalds complained that this cycle was more painful than most, and suggested that the long-term support status may be a part of the problem. A couple of recent pulls into the mainline highlight the pressures that, increasingly, apply to LTS releases. 

As was discussed in [this article](<https://lwn.net/Articles/735887/>), the 4.14 kernel will include some changes to the kernel timer API aimed at making it more efficient, more like contemporary in-kernel APIs, and easier to harden. While API changes are normally confined to the merge window, this change was pulled into the mainline for the 4.14-rc3 release. The late merge has led to a small amount of [grumbling](<https://lwn.net/Articles/735896/>) in the community. 

The problem isn't necessarily the addition of `timer_setup()` which, on its own, cannot really break anything. But that addition has been followed by a series of conversions to the new interfaces, which are being sent to the relevant maintainers. Accepting a `timer_setup()` conversion into a maintainer tree will only work if that tree has `timer_setup()` itself; that implies that the maintainer tree must be current with the mainline as recently as 4.14-rc3. Many subsystem maintainers branch from the mainline around -rc1 or -rc2, so they won't be able to apply the conversion patches unless they perform a separate merge first. The merge is not usually hard, but subsystem trees containing "back merges" with the mainline can [run into trouble](<https://lkml.org/lkml/2011/3/10/538>) during the merge window, so maintainers have understandably become leery of them. 

In this case, the grumbling is already done, and the conversion to the new timer API can be expected to be completed on schedule in 4.15. And, perhaps more to the point, those who want to backport a bunch of conversions to 4.14 (so as to have them in a long-term supported kernel that is likely to be shipped in many mobile devices) will have a much easier task of it. It was never explicitly said that 4.14, in particular, was an important target for this work, but it seems unlikely that it wasn't in developers' minds. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

In another case, things were more explicit. Thomas Gleixner recently [sent a pull request](<https://lwn.net/Articles/735898/>) for a significant refactoring of the watchdog timer subsystem; it was a reworked version of a patch set that had been [refused](<https://lwn.net/Articles/735899/>) by Torvalds during the merge window. Part of the reasoning for requesting a pull this late in the development cycle was a desire to get the work into this release in particular: 

As 4.14 is a long term stable kernel, I prefer to have working watchdog code in that and the lockdep issues resolved. I wouldn't ask you to pull if 4.14 wouldn't be a LTS kernel or if the solution would be easy to backport. 

Stable kernel maintainer Greg Kroah-Hartman [complained](<https://lwn.net/Articles/735900/>) about that request: ""This is exactly what I did _NOT_ want to ever see happen when I did the 'let's announce the LTS kernels ahead of time'"". He suggested that perhaps future long-term support kernels will return to post-release announcements. That notwithstanding, Torvalds pulled the changes for the 4.14-rc4 without comment. 

This work almost certainly will not break the 4.14 kernel; it was essentially ready during the merge window. But it does show that the LTS release is motivating pull requests that might have otherwise waited another cycle. That is not how things were supposed to work; part of the idea behind a nine-week release cycle was that, since the cost of missing one cycle was minimal, there would no longer be any great incentive to hurry code into any particular release. It is clear, though, that this incentive has not entirely gone away; indeed, it may be getting stronger. 

For those who are inside the kernel community, one development cycle looks much like the next. But, for those making use of the kernel, all kernel releases are decidedly not equal. The release that they actually plan to ship is the one that they care about. There has been a determined effort to encourage the industry to ship the LTS kernels in the hope of improving the support for deployed kernels in general. This effort has seen some success, which is a positive change, but it does tend to focus even more attention on the LTS releases. That can only result in more pressure to get features into those releases. 

In a sense, the situation vaguely resembles how things worked before 2.6 came out: major kernel releases were separated by years, so there was immense pressure to get features in before the deadline. As the LTS kernels become more widely used, they start to look like the major releases of old. The LTS releases are the ones that everybody wants to get their features into, and they only happen once each year. Missing an LTS release means waiting a year for a feature to make it into the next LTS release and, probably, maintaining it out-of-tree for products shipped in the meantime. It's not surprising that the idea of getting code into the mainline sooner, even if it requires fixing later, has some appeal, but there is a cost to doing things that way. As Kroah-Hartman said: ""We've been down this path before, and it was not good"". 

That said, the kernel development community has changed considerably since the adoption of the short release cycle. Code is generally of a much higher quality at the time it is merged into the mainline. So if a bit more of it is jostling to get into 4.14, the result may be a more turbulent development cycle. It should not, however, replicate the situation of 15-20 years ago, where a "stable" kernel release would require another year to truly stabilize. We are probably not at risk of repeating the misery of the early 2.x years. 

In the cases described here, the quality of the code being merged is not in question. It is really just a matter of the timing, and the discussion wound down quickly. But this topic can be expected to return. Neither the pressure to get changes into LTS releases nor human desire to game the system will go away, even if the pre-announcement of LTS releases comes to an end.
