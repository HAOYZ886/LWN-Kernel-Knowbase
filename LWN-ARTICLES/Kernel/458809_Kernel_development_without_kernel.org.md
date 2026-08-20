---
title: Kernel development without kernel.org
url: https://lwn.net/Articles/458809/
date: "September 13, 2011"
category: "Development tools-Infrastructure; Kernel.org"
author: "By Jonathan Corbet September 13, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 13, 2011

The security problems at kernel.org have raised concerns about the kernel source and other software hosted there. There has been no evidence, so far, that kernel.org was used to distribute any corrupted software. But there is another aspect to this breakin: kernel.org is "down for maintenance" and there is no word as to when it might come back. As a result, even if no malware was distributed, the kernel.org crack represents a denial of service attack of significant proportions. 

Linus has released two 3.1-rc versions from a temporary site at Github, but there's not a lot of work to be found there. Among other things, the loss of all the repositories hosted on kernel.org means that there is relatively little for him to pull. Stephen Rothwell, meanwhile, continues to pull the trees he can reach to create linux-next. He is able to report integration and build problems, but [cannot put the tree](<https://lwn.net/Articles/458812/>) where others can reach it. ""Besides, I am having a nice restful time."" There have been no stable tree updates since kernel.org went down. 

Alternative trees are beginning to pop up across the net as developers find other places to host their work for now. If the kernel.org outage continues for some time, we can expect to see many more of those show up - though some developers [are refusing](<https://lwn.net/Articles/458914/>) to set up alternative repositories. Most of the substitute trees are described as temporary; it will be interesting to see how many of them actually move back to kernel.org once this episode has run its course. Some developers may decide that keeping their trees elsewhere works better for them. We may have a distributed source control system, but it has become clear that the kernel community works with a rather centralized hosting and distribution infrastructure. 

The loss of kernel.org has slowed things enough to make it clear that the process has a single point of failure built into it. Whether that is worth fixing is not entirely clear; no code should have been lost and, if kernel.org were ever to disappear permanently, the process could be back to full speed on other systems in short order. For now, though, we're seeing things disrupted in a way few other events have been able to manage. It's interesting to ponder on what would have happened had the compromise come out during the merge window.
