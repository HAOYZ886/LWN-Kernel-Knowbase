---
title: "KS2011: Scheduler testing"
url: https://lwn.net/Articles/464296/
date: "October 24, 2011"
category: "Development tools; Scheduler-Testing and benchmarking"
author: "By Jonathan Corbet October 24, 2011 2011 Kernel Summit coverage"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 24, 2011

* * *

[2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>) coverage 

Google's Paul Turner started his session by saying that scheduler testing is a hard problem. Results are hard to reproduce, especially when users cannot share their workloads, and remote debugging is impossible. There are a lot of possible machine topologies, expanding the problem space. The lack of testability makes developers reluctant to make significant changes to the scheduler; it is too easy to break somebody else's workload and not find out until it is too late. But there are a lot of changes that need to be made, especially (for Paul, at least) in the areas of load balancing and power management. What is to be done? 

Load balancing is a concern at Google; having idle CPUs when there are processes waiting for a chance to run on other CPUs is not an ideal situation. It is not too hard to come up with some metrics to describe the "goodness" of the scheduler's load balancing decisions; it is just a matter at looking at the state of the system and determining if a job on one CPU should have been placed on a different one. The problem is that this computation is sufficiently hard that it can't be done in real time; otherwise there would be no real load balancing problem. 

Enter LinSched, a scheduler simulator for Linux. This tool was [released](<https://lwn.net/Articles/409680/>) in 2010, but, unfortunately, "it was awful." The idea was good, but the implementation was not. So the folks at Google have reworked the whole thing, ending up with something that looks a lot like user-mode Linux, but which is aimed at scheduler testing. There are no hooks placed in the scheduler itself, so it is easy to change the scheduler or apply patches before testing. It is fast, and has support for most scheduler features. All of the instrumentation and support has been pushed off into a new "linsched" virtual architecture. 

The next step is to come up with useful workloads. A number of Google workloads were observed to the point where they could come up with patterns for 500 different situations. With the new linsched, they can quickly test all of these workloads with a variety of system topologies; the whole thing can be run under a debugger, so it is easy to stop and examine decisions that go wrong. Finding the "wrong" decisions is a matter of calculating the load-balancing score for a reference scheduler, then looking for places where a test scheduler generates worse scores. This tool has already been used to find (and fix) workload-specific regressions resulting from some mainline scheduler patches. 

They would like to release this code and push it into the kernel's `tools/` directory. That could have even happened for the 3.2 cycle, but the kernel.org outage slowed things down somewhat. Interest in the tool among developers has been high, and in academic communities as well. 

It is safe to say that this news was warmly received in the room at the kernel summit. Having a test tool like this gives developers a much higher degree of confidence when they are making changes to the scheduler. So Ingo Molnar welcomed the tool, asking only that the perf infrastructure be used as much as possible. He would like to see the workload descriptions merged as well so that everything needed to run scheduler tests will be present in the mainline kernel. Scheduler problems, he said, traditionally take a long time to find; having a tool like this would allow the scheduler to be improved much more aggressively. He could see a day when no scheduler patches would be accepted before they had passed this set of tests. 

A number of other things could be added to the tool in the future - simulating preemption latency, for example. It didn't take developers long to say that they would like to have a similar tool for other parts of the kernel - the memory management subsystem came to mind. That is a rather harder problem - that subsystem is rather more complex than the scheduler. But even being able to exhaustively test just the scheduler is a big step in the right direction. 

Next: [Patch review](<https://lwn.net/Articles/464298/>)
