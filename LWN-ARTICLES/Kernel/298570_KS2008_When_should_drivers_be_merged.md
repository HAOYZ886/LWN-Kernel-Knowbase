---
title: "KS2008: When should drivers be merged?"
url: https://lwn.net/Articles/298570/
date: "September 16, 2008"
category: "Development model-Driver merging"
author: "By Jonathan Corbet September 16, 2008 LWN's 2008 Kernel Summit coverage"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 16, 2008

* * *

LWN's [2008 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2008/>)

The rough consensus in the kernel development community over the last couple of years has held that device drivers should be merged into the mainline as soon as possible. Even if these drivers have significant problems, it is better to get them into the mainline where they are more likely to be fixed. This approach was reconsidered at a Kernel Summit 2008 session, but the group left the policy essentially unchanged. 

There are two fundamental lines of thought on this subject. James Bottomley started off the session with his feeling that the time before merging presents the best opportunity to get driver authors to improve their work. The possibility of merging the code provides a motivational incentive which vanishes once the code goes in. So James likes to hold code submissions out of the mainline until the worst problems have been addressed. 

On the other hand, Arjan van de Ven doesn't like the idea of "holding code hostage" in this way. From this point of view, about the only reason to hold drivers out of the mainline is obvious security or user-space API problems. In the absence of those, getting the code merged into the mainline, where it will be more accessible for others to fix, is the best way to improve bad drivers. 

Linus is clearly in the second camp. Drivers which are out of the mainline tree, he says, simply do not get better. People just do not spend much time looking at out-of-tree code. Additionally, not accepting drivers from vendors may put us into a position of having no real traction with those vendors; each of their subsequent drivers will have the same problems. By getting those drivers into the tree and fixing them, we may be able to push them toward producing better code. Otherwise, says Linus, we may be "shooting ourselves in the foot." 

On the other hand, Greg Kroah-Hartman reported some strong successes with his linux-staging tree. That tree currently hosts some 15 drivers, most of which are steadily improving over time. Being in linux-staging is apparently enough to draw some attention to a driver, and that helps to get it into better shape. 

Much of the discussion was devoted to an attempt to set a line dividing drivers which can be merged from those which cannot. There was not a whole lot of success, though. It really appears to be a case-by-case sort of problem. For example, what about one vendor driver which reads a configuration file directly from `/etc`? Such behavior is normally frowned upon. But, if the driver is already out there and being used, putting it into the mainline will not make things worse - we already have the problem. So, especially when the driver is already in widespread use, we might as well just merge it. 

Some ways of mitigating problems with drivers were discussed. Some of the worst behaviors could be configured out, allowing the merging of a barely functional driver which can then be improved in place. Really nasty drivers can set a taint bit in the kernel as a warning to developers trying to track down bugs on the affected systems. Another idea involves outfitting badly-written drivers with strong warnings to keep other developers from copying the code found therein. 

It was suggested that the distributors could ship drivers from the linux-staging tree, perhaps with the taint feature added. The answer to that was that, if the drivers are being shipped by distributors, they might as well be in the mainline. Linus stated that anything the distributors ship should really be merged as well. There are practical difficulties, though; Fedora ships the Nouveau driver, which still has not committed to a stable user-space API. Until that API stabilizes, Nouveau cannot be merged into the mainline, but there is still value in getting the driver tested by Fedora users. 

There were a few conclusions from the discussion. The taint flag for substandard drivers will probably be added. There might be a `drivers/staging` directory for such drivers as well. Greg will take responsibility for getting some of those linux-staging drivers into the mainline; he has, it was suggested, just become the official crap maintainer.
