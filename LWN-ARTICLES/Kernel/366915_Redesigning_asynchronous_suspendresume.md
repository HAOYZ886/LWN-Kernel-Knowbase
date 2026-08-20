---
title: Redesigning asynchronous suspend/resume
url: https://lwn.net/Articles/366915/
date: "December 16, 2009"
category: Development model; Power management
author: "By Jonathan Corbet December 16, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 16, 2009

Your editor suspects that, were somebody to poll the community of Linux users, very few would state that they dislike the idea of having their systems suspend and resume more quickly. Rafael Wysocki has been working toward this goal for some time; his [asynchronous suspend/resume patches](<http://lwn.net/Articles/347573/>) were covered here back in August. This code has not encountered any real turbulence for a while, so one might well assume that Rafael's [2.6.33 pull request](<https://lwn.net/Articles/366916/>) containing asynchronous suspend/resume would not be controversial. Such assumptions, however, fail to take into account the "last-minute Linus" effect. 

The simple fact of the matter is that, like anybody else, Linus cannot possibly follow all of the projects under way at any given time; that makes it entirely possible for work on a specific project to proceed to a conclusion without ever drawing his attention. That will inevitably come to an end, though, when somebody sends a pull request asking that the work be merged into the mainline. It seems clear that some requests are scrutinized more closely than others, but some are looked at closely indeed. The power management request, as it turns out, was one of those. 

Linus [didn't like what he saw](<https://lwn.net/Articles/366920/>), to say the least. The code struck him as overly complex and possibly unsafe; he refused to pull it. In particular, he thought that far too much work went into trying to map out the device tree topology and all of the dependencies between devices. In the past, attempts to make things asynchronous based on just the apparent topology have run into trouble; why should it be different this time? 

Having said that, Linus then went on to outline an alternative solution based mainly on the device tree. In so doing, he wanted to make it possible for most drivers to ignore the concept of asynchronous suspend and resume entirely. For much of the hardware on the system, the time required for either operation is so short that there is really little point in trying to do it in parallel. If a device can be suspended in a few milliseconds, one might as well just do it serially and avoid the complexity. 

For the rest, Linus very much wanted the decision on whether to do things asynchronously to be made at the driver level. But the power management core still needs to know enough about asynchronous operation to wait until it is done; one cannot suspend a controller until all devices connected to it have, themselves, completed suspending. After some revisions, [Linus's plan](<https://lwn.net/Articles/366921/>) came down to something like this: 

  * A reader/writer semaphore (rwsem) is associated with each node in the device tree. These semaphores allow an unlimited number of concurrent reader locks, but only one writer lock can exist at any given time, and writers must first wait for any readers to finish. At the beginning of the suspend process, no locks are taken. 

  * The suspend process is initiated on all children of a given node. If suspend is done synchronously, it happens right away and no further action is required. 

  * Should the driver decide to suspend its device asynchronously, it starts a thread to do that work. It also takes a read lock on the parent's rwsem. 

  * When an asynchronous suspend for a specific device completes, the read lock is released. 

  * The parent node acquires a write lock on its own rwsem before suspending the device. If any child nodes are suspending asynchronously, the write lock will block as a result of the outstanding read locks. Only when all read locks are released - meaning that all children are suspended - can the parent acquire its write lock and suspend. 

For resume, the write lock is taken first, and all children take read locks on their parent before resuming the hardware. That will ensure that all devices complete resuming before any child devices begin the process. 

This scheme has the benefit of simplicity. Getting it implemented took a few rounds of discussion, though, with Linus repeatedly asking developers to retain that simplicity and not try to make up new locking schemes. Things still changed along the way; as of this writing, [the current suspend/resume patch set](<http://lwn.net/Articles/366427/>) does not use Linus's plan as originally written. Among other things, Rafael, who did implement an rwsem-based solution, ran into problems with lockdep that Linus [agreed](<https://lwn.net/Articles/366930/>) were serious. 

What has been implemented instead is a variant on that scheme based on completions. Every device node gets a completion structure, initially set to the "not complete" state. Additionally, any driver which implements asynchronous suspend/resume needs to call `device_enable_async_suspend()` to inform the power management core of that fact. It's now up to that core to create threads for asynchronous suspend/resume operations, and to invoke driver callbacks from those threads. Before suspending a specific device node, the power core will wait for completions for any child devices which have been marked for asynchronous callbacks. Once again, that ensures that all children have been suspended before the parent node is suspended. 

Linus doesn't like the completion-based approach, but has indicated that he will be willing to take it. As of this writing, that has not yet happened, though. 

Seen in one light, this episode highlights the sort of disregard for developer time which is occasionally seen in the kernel development process. It is not that uncommon for code which has seen a lot of work to end up being discarded or massively reworked. This model can seem quite wasteful, and there can be no doubt that it can be highly frustrating for the developers involved. But it is also a fundamental part of how quality control for the kernel works. The suspend/resume code was clearly improved by this last-minute redesign. One might say that it would have been better done some months ago, but what matters most for Linux users is that it happens at all.
