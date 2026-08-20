---
title: Some notes from the BFS discussion
url: https://lwn.net/Articles/351499/
date: "September 9, 2009"
category: Latency; Scheduler
author: "By Jonathan Corbet September 9, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 9, 2009

As was recently [reported](<http://lwn.net/Articles/350100/>) here, Con Kolivas recently resurfaced with [a new CPU scheduler](<http://ck.kolivas.org/patches/bfs/bfs-faq.txt>) called "BFS". This scheduler, he said, addresses the problems which ail the mainline CFS scheduler; the biggest of these, it seems, is the prioritization of "scalability" over use on normal desktop systems. BFS was meant to put the focus back on user-level systems and, perhaps, make the case for supporting multiple schedulers in the kernel. 

[![\[Pipe
benchmark results\]](https://static.lwn.net/images/ns/kernel/bfs-vs-tip-pipe.jpg)](<http://people.redhat.com/mingo/misc/bfs-vs-tip-pipe.jpg>) Since then, CFS creator Ingo Molnar has [responded](<http://lwn.net/Articles/351058/>) with a series of benchmark results comparing the two schedulers. Tests included kernel build times, pipe performance, messaging performance, and an online transaction processing test; graphs were posted showing how each scheduler performed on each test. Ingo's conclusion: ""Alas, as it can be seen in the graphs, i can not see any BFS performance improvements, on this box."" In fact, the opposite was true: BFS generally performed worse than the mainline scheduler. 

Con's [answer](<https://lwn.net/Articles/351504/>) was best described as "dismissive": 

/me sees Ingo run off to find the right combination of hardware and benchmark to prove his point. 

[snip lots of bullshit meaningless benchmarks showing how great cfs is and/or how bad bfs is, along with telling people they should use these artificial benchmarks to determine how good it is, demonstrating yet again why benchmarks fail the desktop] 

As far as your editor can tell, Con's objections to the results mirror those heard elsewhere: Ingo chose an atypical machine for his tests, and those tests, in any case, do not really measure the performance of a scheduler in a desktop situation. The more cynical observers seem to believe that Ingo is more interested in defending the current scheduler than improving the desktop experience for "normal" users. 

The machine chosen was certainly at the high end of the "desktop" scale: 

So the testbox i picked fits into the upper portion of what i consider a sane range of systems to tune for - and should still fit into BFS's design bracket as well according to your description: it's a dual quad core system with hyperthreading. It has twice as many cores as the quad you tested on but it's not excessive and certainly does not have 4096 CPUs. 

A number of people thought that this box is not a typical desktop Linux system. That may indeed be true - today. But, as Ingo (among others) has [pointed out](<http://lwn.net/Articles/351193/>), it's important to be a little ahead of the curve when designing kernel subsystems: 

But when it comes to scheduler design and merge decisions that will trickle down and affect users 1-2 years down the line (once it gets upstream, once distros use the new kernels, once users install the new distros, etc.), i have to "look ahead" quite a bit (1-2 years) in terms of the hardware spectrum. 

Btw., that's why the Linux scheduler performs so well on quad core systems today - the groundwork for that was laid two years ago when scheduler developers were testing on a quads. If we discovered fundamental problems on quads _today_ it would be way too late to help Linux users. 

Partly in response to the criticisms, though, Ingo [reran his tests](<https://lwn.net/Articles/351509/>) on a single quad-core system, the same type of system as Con's box. The end results were just about the same. 

The hardware used is irrelevant, though, if the benchmarks are not testing performance characteristics that desktop users care about. The concern here is latency: how long it takes before a runnable process can get its work done. If latencies are too high, audio or video streams will skip, the pointer will lag the mouse, scrolling will be jerky, and Maelstrom players will lose their ships. A number of Ingo's original tests were latency-related, and he added a couple more in the second round. So it looks like the benchmarks at least tried to measure the relevant quantity. 

Benchmark results are not the same as a better desktop experience, though, and a number of users are reporting a "smoother" desktop when running with BFS. On the other hand, making significant scheduler changes in response to reports of subjective "feel" is a sure recipe for trouble: if one cannot measure improvement, one not only risks failing to fix any problems, one is also at significant risk of introducing performance regressions for other users. There has to be some sort of relatively objective way to judge scheduler improvements. 

The way preferred by the current scheduler maintainers is to identify causes of latencies and fix them. The kernel's infrastructure for the identification of latency problems has improved considerably over the last year or two. One useful tool is [latencytop](<http://www.latencytop.org/>), which collects data on what is delaying applications and presents the results to the user. The ftrace tracing framework is also able to create data on the delay between when a process is awakened and when it actually gets into the CPU; see [this post from Frederic Weisbecker](<https://lwn.net/Articles/351513/>) for an overview of how these measurements can be taken. 

If there are real latency problems remaining in the Linux scheduler - and there are enough "BFS is better" reports to suggest that there are - then using the available tools to describe them seems like the right direction to take. Once the problem is better understood, it will be possible to consider possible remedies. It may well be that the mainline scheduler can be adjusted to make those problems go away. Or, possibly, a more radical sort of approach is necessary. But, without some understanding of the problem - and associated ability to measure it - attempted fixes seem a bit like a risky shot in the dark. 

Ingo welcomed Con back to the development community and invited him to help improve the Linux scheduler. This seems unlikely to happen, though. Con's way of working has never meshed well with the kernel development community, and he is showing little sign of wanting to change that situation. That is unfortunate; he is a talented developer who could do a lot to improve Linux for an important user community. The adoption of the current CFS scheduler is a direct result of his earlier work, even if he did not write the code which was actually merged. In general, though, improving Linux requires working with the Linux development community; in the absence of a desire to do that effectively, there will be severe limits on what a developer will be able to accomplish. 

(See also: [Frans Pop's benchmark tests](<https://lwn.net/Articles/351636/>), which show decidedly mixed results.)
