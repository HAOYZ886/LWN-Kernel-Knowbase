---
title: "ELC: A PREEMPT_RT roadmap"
url: https://lwn.net/Articles/440064/
date: "April 27, 2011"
category: Realtime
author: "By Jake Edge April 27, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
April 27, 2011

Thomas Gleixner gets asked regularly about a "roadmap" for getting the realtime Linux (aka PREEMPT_RT) patches into the mainline. As readers of LWN will know, it has been a multiple-year effort to move pieces of the realtime patchset into the mainline--and one that has been [predicted](<https://lwn.net/Articles/313045/>) to complete several times, though not for a few years now. Gleixner presented an update on the realtime patches at this year's Embedded Linux Conference. In the talk, he showed a roadmap--of sorts--but more importantly described what is still lurking in that tree, and what approach the realtime developers will be taking to get those pieces into the mainline. [ ![\[Thomas Gleixner\]](https://static.lwn.net/images/2011/elc-gleixner-sm.jpg) ](<https://lwn.net/Articles/440239/>)

Gleixner started out by listing the parts of the realtime tree that have already made it into the mainline. That includes high-resolution timers, the mutex infrastructure, preemptible and hierarchical RCU, threaded interrupt handlers, and more. Interrupt handlers can now be forced to run as threads by using a kernel command line option. There have also been cleanups done in lots of places to make it easier to bring in features from the realtime tree, including cleaning up the locking namespace and infrastructure ""so that sleeping spinlocks becomes a more moderate sized patch"", he said. 

#### Missing pieces

What's left are the ""tough ones"" as all of the changes that are ""halfway easy to do"" are already in the mainline. The next piece that will likely appear is the [preemptible mmu_gather](<https://lwn.net/Articles/436785/>) patches, which will allow much of the memory management code to be preemptible. Gleixner said that it was hoped that code could make it into 2.6.39; that didn't happen, but it should go in for 2.6.40. 

Per-CPU data structures are a current problem that ""makes me scratch my head a lot"", Gleixner said. The whole idea is to keep the data structures local to a particular CPU and avoid cache contention between CPUs, which requires that any code modifying those data structures stay running on that CPU. In order to do that, the code disables preemption while modifying the per-CPU data. If that code ""just did a little fiddling"" with preemption disabled, it would not be a problem, but currently there are often thousands of lines of code executed. The realtime developers have talked with the per-CPU folks and they ""see our pain"". The right solution is use inline functions to annotate the real atomic accesses, so that the preemption-disabled window can be reduced. ""Right now, there is a massive amount of code protected by `preempt_disable()`"", he said. 

The next area that needs to be addressed is preemptible memory and page allocators. Right now, the realtime tree uses SLAB because the others are ""too hard to deal with"". There has been talk about creating a memory allocator specifically for the realtime tree, but some recent developments in the SLUB allocator may have removed the need for that. SLUB has been converted to be completely lockless for the fast path and Christoph Lameter has promised to deal with the slow path, which is ""good news"" for the realtime developers. The page allocator problem is ""not that hard to solve"", Gleixner said. Some developers have claimed that a fully preemptible, lockless page allocator is possible, so he is not worried about that part. 

Another area ""that we still have to twist our brain around"" is software interrupts, he said. Those currently disable preemption, but then can be interrupted themselves, leading to unbounded latency. One possibility is to split up the software interrupts into different threads and to wake them up when an interrupt is generated, whether it comes from kernel or user space. There are performance implications with that, however, because there is a context switch associated with the interrupt. There are some other ""nasty implications"" as well, because it will be difficult to tune the priorities of the interrupt threads correctly. 

Another possibility would be to add an argument to `local_bh_disable()` that would indicate which software interrupts should be held off. But cleaning up the whole tree to add those new arguments is ""nothing I can do right now"", he said. There are tools to help with adding the argument itself, but figuring out which software interrupts should be disabled is a much bigger task. 

The ""last thing"" that is still pending in the realtime tree is sleeping spinlocks. That work is fairly straightforward he said, only requiring adding one file and patching three others. But that will only come once the other problems have been solved, he said. 

#### Mainline merging

So, when will the merge to mainline be finished? That's a question Gleixner and the other realtime developers have been hearing for seven years or so. The patchset is huge and ""very intrusive in many ways"", he said. It has been slowly getting into the mainline piece by piece, but it will probably never be complete, because people keep coming up with new features at roughly the same rate as things move into the mainline. As always, Gleixner said, ""it will be done by the end of next year"". 

Gleixner used a 2010 [quote from Linus Torvalds](<https://lwn.net/Articles/370998/>) (""The RT people have actually been pretty good at slipping their stuff in, in small increments, and always with good reasons for why they aren't crazy."") to illustrate the approach taken by the realtime developers. The realtime changes are slipped into ""nice Trojan horses"" that are useful for more than just realtime. Torvalds is ""well aware that we are cheating, but he doesn't care"" because the changes fix other problems as well. 

The realtime tree has been pinned to kernel 2.6.33 for some time now (with 2.6.33.9-rt having been released just prior to Gleixner's talk). There are plans to update to 2.6.38 soon. There a several reasons why the realtime tree is not updated very rapidly, starting with a lack of developer time. The tree also requires a long stabilization phase, partly because ""some of the bugs we find are very complex race conditions"", and those bugs can have serious impacts on filesystems or other parts of the kernel. Typically the problem is not fixing those kinds of bugs, but finding them as they can be quite hard to reproduce. 

Another problem is that because the realtime changes aren't in the mainline Gleixner ""can't yell at people yet"" when they break things. Also, other upstream work and merging other code often takes priority over work in the realtime tree. But he is ""tired of maintaining that thing out of tree"", so work will progress. Often getting a piece of the realtime tree accepted requires lots of work elsewhere in the tree, which consumes a lot of time and brain power. ""People ship crap faster than you can fix it"", he said. 

There are about 20 active contributors to the realtime tree, as well as large testing efforts going on at Red Hat, IBM, OSADL, and Gleixner's company Linutronix. 

Looking beyond the current code, Gleixner outlined two potential future features. The first is non-priority-based scheduling, which is needed to solve certain kinds of problems, but brings with it a whole new set of problems. Even though priorities are not used, there are still ""priority-inversion-like problems"" that will have to be solved with mechanisms similar to priority inheritance. Academics have proved that such schedulers can work on uni-processor systems, but have just now started to ""understand that there is this thing called SMP"". Though there is a group in Pisa, Italy (working on [deadline scheduling](<https://lwn.net/Articles/356576/>)) that Gleixner specifically excluded from his complaints about academic researchers. 

The other new feature is CPU isolation, which is not exactly realtime work, but the realtime developers have been asked to look into it. The idea is to hand over a CPU to a particular task, so that it gets the full use of that CPU. In order to do that, the CPU must be removed from the timer interrupt and the RCU pool among other things. The problem isn't so much that users want to be able to run undisturbed for an hour on a CPU or core, but that they then want to be able to interact with the rest of the kernel to send data over the network or write to disk. In general, it's fairly clear what needs to be done to implement CPU isolation, he said. 

#### Roadmap

![\[RT roadmap\]](https://static.lwn.net/images/2011/elc-rt-roadmap.png)

It is obvious that Gleixner is tired of being asked for a roadmap for the realtime patches. Typically it isn't engineers working on devices or other parts of the kernel who ask for it, but is, instead, their managers who are looking for such a thing. There are several reasons why there is no roadmap, starting with the fact that kernel developers don't use PowerPoint. More seriously, though, the realtime developers are making their own road into the kernel, so they are looking for a road to follow themselves. But, so that it could no longer be said that he hadn't shown a roadmap, Gleixner presented one (shown at right) to much laughter. 

He also fielded quite a few audience questions about the realtime tree, what others can do to help it progress, and why some of the troublesome Linux features couldn't be eliminated to make it easier to get the code merged. In terms of help, the biggest need is for more testing. In particular, Gleixner encouraged people to test the realtime patches atop Greg Kroah-Hartman's 2.6.33 stable series. 

Software interrupts are still required in various places in the kernel, in particular the network and block layers. Any change to try to remove them would require changes in too much code. On the other hand, counting semaphores are mostly gone, though some uses come in through the staging tree. Those are mostly cleaned up before the staging code moves out of that tree, he said. From time to time, he looks through the staging tree for significant new users of counting semaphores and doesn't really find any, so he is not concerned about those, but is more concerned about read-write semaphores. 

As for the choice of 2.6.38 as the basis for the next realtime tree, Gleixner said that he picks the ""most convenient"" tree when making that decision. It depends on what is pending for the mainline, and what went into the various kernel versions, because he does not want to backport things into the realtime tree: ""I'm not insane"", he said. 

The realtime tree got started partially because of a conference he attended in 2004 where various academics gathered there agreed that it was not possible to turn a general purpose operating system into a realtime one. He started working on it because of that technical challenge. Along the same lines, when asked what he would do with all the free time he would have once the realtime code was upstream, Gleixner replied that he would like to eliminate jiffies in the kernel. He has a ""strong affinity to mission impossible"", he said. 

One should be careful about choosing the realtime kernel and only use it if you need the latency guarantees, he said. So smartphone kernels might not have any real need for such a kernel, he said. But if the baseband stack were to move to the main CPU, then it might make sense to look at using the realtime code. One ""should only run such a beast if you really need it"". That said, he rattled off a number of different projects that were using the realtime kernel, including military, banking, and automation applications. He closed with a short description of a gummy bear sorting machine that used the realtime kernel, and was quite fancy, but after watching it for a bit, you wouldn't want to see gummy bears again for a year.
