---
title: "KS2010: A staging process for ABIs"
url: https://lwn.net/Articles/412744/
date: "November 2, 2010"
category: "Development model-User-space ABI"
author: "By Jonathan Corbet November 2, 2010 2010 Kernel Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 2, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

Like many kernel developers, Steve Rostedt has found out that user-space interfaces are hard. API design is hard in general, but, once an interface makes it into a released kernel it must be maintained indefinitely. Breaking applications is just not something that can be allowed to happen. But we always learn things about ABIs after people start trying to actually use them. So, he asked: do we need a way to stage new ABIs into the kernel? New interfaces, perhaps, could be specially marked and only available via debugfs; they could be withdrawn or changed in any future kernel release. That scheme would give developers a chance to find and fix any remaining problems before committing to the ABI. 

The answer from Linus came quickly: "no." Any sort of staging process for adding ABIs would, in his opinion, be a failure. If we need any such thing, we are clearly just adding too many ABIs in the first place. We have too many system calls, and too many other ways of interacting with the kernel. We should, instead, be talking about how to say "no" more often. 

Another way of putting it, he said, is that, if you still want people to try out an ABI, you should not be asking him to pull it. In general, it can be better if new interfaces stay out of the kernel for a while. SystemTap was given as an example here: according to Linus, time has shown that the SystemTap interface is not a good one. He's very glad he never pulled it into the kernel. The lesson is that it's a good idea to impose a certain amount of pain on people who want to create new interfaces; let them live out of the mainline for a while. If, after five years it looks like a good idea, the code can be taken upstream. Maintainers should not be accepting ABIs which have not seen that sort of testing. 

Ted Ts'o talked for a bit about the financial resources behind some new features; fanotify was given as an example. Companies that want such features will put their cash behind them; given enough time, some of those features will get past the community's defenses. It is going to happen at times, how can we deal with it? 

Another example that was raised was the Android suspend blockers. The answer here is that the code has now been merged; the final pieces went in for 2.6.37. Of course, it's not suspend blockers that were merged, it was the opportunistic suspend and wakeup sources work done by Rafael Wysocki; suspend blockers "done the right way." The only problem here is that the Android developers have not said whether they will use this ABI or not; this particular interface is essentially untested and without users at the moment. 

Should new ABIs go into linux-next? Patches going there are supposed to be intended for merging in the next development cycle, so the answer is "no" unless it's clear that the ABI is ready to go in. 

What about removing ABIs that we don't like? Linus's response was, once again, clear: breaking applications is a regression. So if he gets even a single complaint about a removed interface, he'll revert the patch and put the ABI back. It's really only possible to remove an ABI if nobody will notice that it's gone. Andrew Morton agreed, but also pointed out that we have to have some way of getting rid of old ABIs if we are going to preserve our sanity over the long term. The first step, he says, is to warn users. Then, after a while (five to ten years, perhaps), there will be no users left and the code can go away. Linus noted that Google can be an effective way of looking for deprecation warnings. If nobody has posted a log with a warning in at least a year, it's probably safe to remove the interface. 

Andrew added that a bad ABI indicates a failure of the review process. And the review process, he said, is what we should be caring about more than anything else. When a new ABI is posted, everybody should be looking at it. He was clearly not happy about the amount of review that is happening now. 

Dave Airlie asked if it would help to require man pages for every new ABI. In the past, Michael Kerrisk's man page work has helped to reveal a number of ABI problems and bugs, but Michael is not doing that anymore. Linus responded that we've tried in the past, but it hasn't worked very well. Al Viro added that "the man page kind of sucks" is a weak last line of defense which comes too late. Beyond that, as Linus noted, man pages tend to describe system calls, but that's not where the real problem is. Much more ABI trouble comes from tracepoints, `ioctl()` calls, sysfs, etc. 

Ted said that, in the end, bad ABIs are really a maintainer problem. Maintainers have to say "no" more often. Hugh Dickins suggested that a special effort could be dedicated to removing crap in the -rc2 and -rc3 releases that was added in -rc1. At the closing of the session, it was suggested that there would be value in having a tool which could identify all new user-space ABIs added since the previous kernel release. That could make a good project for somebody who would like to help the kernel process. 

[Next: Deadline scheduling](<https://lwn.net/Articles/412745/>).
