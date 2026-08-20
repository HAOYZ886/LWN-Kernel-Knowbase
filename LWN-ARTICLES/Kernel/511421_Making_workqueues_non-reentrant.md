---
title: "Making workqueues non-reentrant"
url: https://lwn.net/Articles/511421/
date: "August 15, 2012"
category: Workqueues
author: "By Jonathan Corbet August 15, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 15, 2012

Workqueues are the primary mechanism for deferring work within the Linux kernel. The maintainer of this facility is Tejun Heo, who recently posted [a patch series](<https://lwn.net/Articles/511190/>) changing an aspect of workqueue behavior that, perhaps, few kernel developers know about. Most workqueues are reentrant, meaning that the same work item can be running on multiple CPUs at the same time. That can result in concurrency that developers are not expecting; it also significantly complicates the various "flush" operations in surprising ways. In summary, Tejun says: 

In the end, workqueue is a pain in the ass to get completely correct and the breakages are very subtle. Depending on queueing pattern, assuming non-reentrancy might work fine most of the time. The effect of using flush_work() where flush_work_sync() should be used could be a lot more subtle. flush_work() becoming noop would happen extremely rarely for most users but it definitely is there. 

A tool which is used as widely as workqueue shouldn't be this difficult and error-prone. This is just silly. 

Tejun's patch set is intended to make the workqueue interface work more like the timer interface, which is rather more careful about allowing concurrent operations on the same timer. All workqueues become non-reentrant, and aspects of the API related to reentrant behavior have been simplified. There is, evidently, a slight performance cost to the change, but Tejun says it is small, and the cost of flush operations should go down. It seems like a worthwhile trade-off, overall, but anybody who maintains code that depends on concurrent work item execution will want to be aware of the change.
