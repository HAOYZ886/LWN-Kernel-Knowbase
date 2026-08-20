---
title: Braking CPU hotplug
url: https://lwn.net/Articles/598460/
date: "May 14, 2014"
category: Hotplug
author: "By Jonathan Corbet May 14, 2014"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 14, 2014

As a normal rule, kernel developers do not like papering over problems or putting in hacks to hide those problems. Though a recently posted [patch](<https://lwn.net/Articles/598462/>) to apply just such a hack seems unlikely to actually be merged, it was generally well received and does demonstrate an interesting use of this technique. When all hope for a proper fix to a problematic subsystem appears to be lost, perhaps it's time for desperate measures. 

The subsystem in question is the one that handles CPU hotplugging. There are a number of reasons why one might want to add a CPU to (or remove a CPU from) a running system: the hardware may actually support physical addition or removal of CPUs, or one might want to offline a misbehaving processor. In the virtualization world, CPU hotplugging is an obvious way to adjust the processing power available to specific guests as they run. The feature clearly has value, and nobody would seriously suggest removing it. But nobody is happy with how CPU hotplugging has been implemented. 

Changing the set of CPUs in a running system is a complex task; there is a vast amount of per-CPU state at numerous levels that must be managed. In such a situation, it makes sense to have an overall mechanism that manages the complexity, breaks it down into simple steps, and ensures that those steps are run in the correct order. Unfortunately, the Linux kernel does not have that sort of mechanism; instead, it has a confusing array of notifiers and callbacks that is hard to reason about or make changes to. And, unsurprisingly, it has bugs; developers who have gone looking for bugs in this area have had little trouble finding them. 

In fact, bugs in this area are so plentiful that Borislav Petkov wants to make them harder to find. His patch introduction reads: 

We have all those eager tester dudes which scratch up a dirty script to pound on CPU hotplug senselessly and then report bugs they've managed to trigger. 

Well, first of all, most, if not all, bugs they trigger are CPU hotplug related anyway. But we know hotplug is full of duct tape and brown paper bags. So we end up clearly wasting too much time dealing with a mechanism we know it is b0rked in the first place. 

His solution was simple: insert a one-second delay into each CPU hotplug operation. Slowing things in that way minimizes the number of operations that can be tested and the amount of concurrency between operations. It should reduce the flow of bug reports nicely. 

There is one tiny little problem, of course: this patch does not actually fix any bugs; it just hides them from view. Andrew Morton was quick to [point out](<https://lwn.net/Articles/598466/>) that this patch would almost certainly result in fewer CPU hotplug bugs being fixed. But Thomas Gleixner [thinks that may be a good thing](<https://lwn.net/Articles/598468/>): ""if people would have spent the same amount of time to rewrite the hotplug mess, we would have a way bigger benefit. But no, we prefer to add more layers of duct tape and bandaid hackery to it."" 

Thomas, of course, has in the past tried to do just that sort of rewrite; that work was [covered here](<https://lwn.net/Articles/537562/>) in February 2013\. He took the time to break the hotplug and hotunplug operations down into a long list of discrete steps; he then built a system to run those steps in a well-defined order. It was a far cry from a full solution to the problem; most of the existing hotplug code remained in place and was just called differently. But it provided a framework on which a more complete rewrite could be done over time. 

The only problem is: nobody did that rewrite. Thomas ran out of time and moved back to other tasks, and nobody else picked up the work, so the patches have languished since their initial posting. The work that has been done in that area, instead, is the application of increasingly complex bug fixes ([recent example](<https://lkml.org/lkml/2014/5/6/475>)) as problems turn up and developers try to make the existing implementation work. These fixes may address specific bugs, but they do not address the complexity and unmaintainability of the system as a whole; indeed, they tend to make those problems worse. 

It is frustration with the addition of more "duct tape and brown paper bags" that led to Borislav's patch to slow the hotplug system down. In the end, the developers who have to work with this part of the code don't _want_ more bug fixes; they want the code to be made simpler and easier to understand so that, in the end, there will not be a need for an endless stream of fixes that just add more complexity to the code. Making it harder to find bugs in this subsystem is a heavy-handed way of trying to direct developers' attention elsewhere. 

Naturally, the patch is more of a statement than a serious attempt to change the kernel; it would be surprising if this patch were merged. In a world where kernel subsystem maintainers cannot force developers to work on a specific area, and where no company managers have seen fit to direct their employees to solve the CPU hotplug problem, one has to be creative sometimes to get things done. One might hope that this patch posting would be a strong enough hint to get somebody to work on the problem. Unfortunately, Thomas may have inadvertently sabotaged that effort by saying that, if nobody else gets around to rewriting the hotplug CPU subsystem, he will jump back in and do it himself.
