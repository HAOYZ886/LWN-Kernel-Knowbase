---
title: "KS2010: Performance regressions"
url: https://lwn.net/Articles/412747/
date: "November 2, 2010"
category: Performance regressions
author: "By Jonathan Corbet November 2, 2010 2010 Kernel Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 2, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

There have been many grumblings over the years that kernel releases are often slower than their predecessors. One often hears nostalgic longings for the good old days of 2.6.26, 2.6.18, or even older kernels. In a session with no formal leaders, the Kernel Summit attendees were asked to bring out their "war stories" for discussion. 

Paul Turner, who works on Google's kernel team, started off. Over time, Google has had problems with a number of aspects of kernel performance. Out-of-memory handling is an ongoing source of pain. The CFS scheduler, he said, was "merged somewhat aggressively" and caused significant performance regressions for them. Google has a couple of difficulties when it comes to dealing with performance regressions, one of which is the company's habit of jumping about six kernel versions at a time. That makes it hard to catch problems in a timely way or to figure out how they were introduced. Google is trying harder to stay up with the mainline, though, so the latest jump was from 2.6.34 to 2.6.36. There are indeed some new performance regressions between the two. Google's other problem is that it has no way to demonstrate its workload to the kernel community. So kernel developers cannot see Google's performance regressions or know when they have been fixed. 

Linus said that identifying that a problem was introduced between 2.6.34 and 2.6.36 is still too broad an interval. He requested that Google dedicate a couple of machines to running its workloads on daily snapshots. When a performance regression can be narrowed down to a single day's patches, it is a lot easier to find. Google's Mike Rubin agreed with all of this, saying that he would like to set up a group of machines running normal hardware (instead of Google's special systems) and well-known benchmarks, with the information publicly available. Arjan van de Ven noted that Intel is already doing that kind of work. 

Mike also said that a lot of performance regressions tend to come in through the virtual filesystem layer. He also said that Google is seeing some serious scalability problems there; he would like to see Nick Piggin's VFS scalability work merged soon. 

How should performance regressions be reported? The best thing, of course, is a bisection which fingers the guilty commit. Doing that requires highly repeatable tests, though; if a performance benchmark has a 5% variation between runs, it cannot be used for bisection. Paul said that Google has had good results using [linsched](<https://lwn.net/Articles/409680/>) for performance testing. 

Mike wondered: what do maintainers use to spot performance regression? Linus responded: "users." Steve Rostedt chimed in with a plug for his recently-posted [ktest](<https://lwn.net/Articles/412302/>) script. The real answer, though, appears to be that a lot of the real performance testing and fixing is done by distributors when they are working on a new enterprise kernel release. 

It was noted that tracking down performance regressions can be a problem. There is rarely a single bug which slows things by 5-15%; instead, there is a series of 0.5% regressions which all add up. They can be hard to find, especially given that little things like the size or layout of the kernel image can affect things on that scale. Paul noted that, in one case, adding three no-op instructions to the end of one function increased performance by 1.5%. 

As a result, James Bottomley said, kernel developers tend to let a lot of minor regressions pile up over time. Then the distributors need to get an enterprise kernel out, so they put considerable resources into fixing these regressions, one at a time. There is no real pooling of information; each distributor works independently to make things faster. Ted Ts'o said that each distributor tends to have a collection of customer workloads which was obtained under non-disclosure agreements; these workloads are run late in the process, and any resulting regressions are fixed then. Those workloads \- and information about them - cannot be shared. 

Other kinds of testing include The Well Known Database Benchmark Which Cannot Be Named. It can yield useful results, but it can also take a week to run. That, it was dryly noted, can make bisection an even more painful process than usual. 

James asked: should the kernel community care about small performance regressions? After all, there are people out there with big machines, the resources to run benchmarks on them, and the motivation to submit fixes. Mike Rubin said that, as long as there is no credible competitor to Linux, the kernel community maybe doesn't have to care. Ted said that, if the community did care more, it might help to get this big users to update their kernels more often. 

Is there a need for a benchmark summit, a place where kernel maintainers can share performance data? Ted said a good start might be to just post results which can be shared. Such a summit might be scheduled; if so, it will probably be associated with the Linux Foundation's Collaboration Summit in April. 

[Next: Big out-of-tree projects](<https://lwn.net/Articles/412748/>).
