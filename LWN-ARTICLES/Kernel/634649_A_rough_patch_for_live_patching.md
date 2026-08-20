---
title: A rough patch for live patching
url: https://lwn.net/Articles/634649/
date: "February 25, 2015"
category: Live patching
author: "By Jonathan Corbet February 25, 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
February 25, 2015

One of the headline features in the upcoming 4.0 kernel is live patching — the ability to apply a patch to a running kernel and fix a problem without disrupting the operation of the system. The truth of the matter, though, is that the live-patching support merged for 4.0 is only the beginning of the story; quite a bit more work will have to be done to have full support for this feature in the kernel. And now it seems that this work may take a bit longer than the developers involved had hoped; indeed, one prominent developer is calling for the entire concept to be rethought. 

The code merged for 4.0 is a common core that is able to support patching with both [kpatch](<https://lwn.net/Articles/597407/>) and [kGraft](<https://lwn.net/Articles/596854/>). It provides an API that allows patch-containing modules to be inserted into the kernel; it also allows the listing and removal of patches if need be. This API performs the low-level redirection needed to replace patched functions. That is good as far as it goes, but it is missing an important component, called the "consistency model," that ensures the safety of switching between versions of a function in a running kernel. If the change is simple, it may be possible to safely make the change at any time. More complicated changes, though, may require that no kernel code is running in any of the affected functions before the switch can be done. The consistency model as found in kpatch and kGraft is where some of the biggest differences between those two implementations lie, so some work will clearly be needed to bring them together. 

As originally developed, kpatch worked by calling `stop_machine()` to bring the entire system to a halt. It then would check the stack of every process in the system to ensure that none are running within the function(s) to be patched; if the affected functions are not currently running, the patch can proceed, otherwise the operation fails. KGraft, instead, used a "two-universe" model where every process in the system is switched from the old code to the new at a "safe" point. The most common safe point is exit from a system call; at that point, the process cannot be running in any kernel code. 

#### A unified consistency model

Both approaches have their advantages and disadvantages; an attempt to unite them would, hopefully, take the best from each. And that is what Josh Poimboeuf tried to do with his [consistency model patch set](<https://lwn.net/Articles/632582/>) posted in early February. This approach retains the two-universe model from kGraft, but it uses the stack-trace checking from kpatch to accelerate the task of switching processes to the new code. In theory, this technique increases the chances of successfully applying patches while doing away with kpatch's disruptive `stop_machine()` call and much of kGraft's higher code complexity. 

The first objections to be raised focused on one particular aspect of the consistency code: the stack check. As Peter Zijlstra [put it](<https://lwn.net/Articles/634653/>): 

So far stack unwinding has basically been a best effort debug output kind of thing, you're wanting to make the integrity of the kernel depend on it. You require an absolute 100% correctness of the stack unwinder -- where today it is; as stated above; a best effort debug output thing. That is a _big_ change. 

Ingo Molnar also [came out against](<https://lwn.net/Articles/634654/>) the use of stack traces. It comes down to the fact that getting a reliable stack trace out of a process running in kernel space is not as easy as one might expect. There have been lots of bugs in that code in the past, and each architecture brings its own set of special glitches to deal with. And, as Ingo pointed out: 

More importantly, there's no strong force that ensures we can rely on stack backtraces: correcting bad stack traces depends on people hitting those functions and situations that generate them, seeing a bad stack trace, noticing that it's weird and correcting whatever code or tooling quirk causes the stack entry to be incorrect. 

What that means is that a bug in the traceback code is quite likely to stay out of sight until some distributor issues a live patch, at which point things will go badly wrong. The idea of things going badly wrong and disrupting a running system is just what users calling for live patching are most wanting to avoid, so one can imagine that widespread unhappiness would ensue. But it is a risk that will always be hard to avoid, since the correct functioning of the kernel does not otherwise depend on perfectly accurate stack traces. 

There are a number of approaches to consistency, and not all of them use stack traces. Given the opposition to that idea, it seems likely that future proposals will omit that technique. But that leaves open the question of what _will_ be used. Ingo is [pushing strongly](<https://lwn.net/Articles/634660/>) for an approach that forces every process in the system into a quiescent, non-kernel state before applying a patch. It is arguably the simplest approach; it also puts the kernel in a state where it is easy to know that applying the patch is a safe thing to do. 

But, as it turns out, the "simplest" approach still has a fair number of tricky details. Kernel threads cannot be pushed out of kernel space, so some other solution must be found for them. Processes that are blocked in the kernel for some sort of long-term wait need to be unblocked, preferably in a way that can be restarted transparently once the patching process is complete. That could require changes to the implementation of a lot of system calls — and, perhaps, a lot of drivers as well. Some ideas for simplifying this task have circulated, but it would take a while to get an implementation to the point where it would reliably succeed in patching a running kernel. 

An alternative would be to just go with the kGraft two-universe model, which does not depend on stack traces. The downside with this approach is that the process of trapping every process in a safe place can take an unbounded period of time during which the system is in a weird intermediate state. Yet another alternative is to do without the consistency model entirely. That would severely limit the range of patches that could be applied, but it seems that most security fixes (involving, say, the addition of a simple range check) could still be applied to a running system. 

#### Live kernel upgrades

Perhaps feeling that he had not stirred the anthill sufficiently, Ingo went on to [propose](<https://lwn.net/Articles/634663/>) giving up on both kpatch and kGraft, saying ""I think they are fundamentally misguided in both implementation and in design, which turns them into an (unwilling) extended arm of the security theater"". Rather than trying to patch a running kernel, he suggested, why not just save the entire state of the system, boot into an entirely new kernel, then restore the previous state on top of the new kernel? That would get rid of consistency models, greatly expand the range of patches that can be applied, and, in theory, would be more robust. 

This idea is not new, of course. The developers working on CRIU (checkpoint-restore in user space) have had [seamless kernel upgrades](<http://criu.org/Usage_scenarios#Seamless_kernel_upgrade>) in their list of use cases for a while, and they evidently have it working for some workloads. But making this functionality work robustly on all systems would require a great deal of extra work to snapshot the full system state (including the state of devices) and restore it all under an arbitrarily different kernel. Vojtech Pavlik, one of the developers behind kGraft, [estimated](<https://lwn.net/Articles/634665/>) that it would take ten years to make such a system work. 

The users asking for live patching, it is safe to say, would not be thrilled about the prospect of waiting that long. It is also far from clear that the full-upgrade technique, once it actually works, can ever be fast enough to keep those users happy. Ingo estimated that a live upgrade could complete within ten seconds, but that is an eternity to users who find even subsecond stalls for patching to be overly disruptive. So, while there is widespread agreement that live upgrades are an interesting and possibly useful technology, there is little chance that any of the developers currently working on live patching will decide to refocus their efforts on live upgrades. 

So work on live patching will continue, but it is not clear what direction that work will take. The hopes of getting the consistency-model code ready for the 4.1 merge window now seem somewhat remote; getting consensus on a design that can be merged could take some time. So, while it is still possible that the kernel will have an essentially complete live-patching feature by the end of the year, it may happen rather closer to the end of the year than the developers involved might have hoped for.
