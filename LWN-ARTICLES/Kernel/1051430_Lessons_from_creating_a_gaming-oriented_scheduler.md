---
title: "Lessons from creating a gaming-oriented scheduler"
url: https://lwn.net/Articles/1051430/
date: "January 7, 2026"
category: "BPF-CPU scheduling; Scheduler-Extensible scheduler class"
author: "By Jake Edge January 7, 2026 LPC"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
January 7, 2026

* * *

[LPC](<https://lwn.net/Archives/ConferenceIndex/#Linux_Plumbers_Conference-2025>)

At the [2025 Linux Plumbers Conference](<https://lpc.events/event/19/>) (LPC), held in Tokyo in mid-December, Changwoo Min led a [session](<https://lpc.events/event/19/contributions/2150/>) on what he has learned while developing the "[latency-criticality aware virtual deadline](<https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_lavd#scx_lavd>)" (LAVD) scheduler, which is aimed at gaming workloads. The session was part of the [Gaming on Linux](<https://lpc.events/event/19/sessions/225/#20251213>) microconference, which is a new entrant into LPC; organizers hope to see it return [next year in Prague](<https://lpc.events/event/20/>) and, presumably, beyond. LAVD uses the [extensible scheduler class](<https://lwn.net/Articles/922405/>) (sched_ext) and has the primary goal of minimizing [stuttering](<https://www.gameslearningsociety.org/what-is-game-stuttering/>) in games; it is implemented in a combination of BPF and Rust. 

Min said that he has been developing LAVD as part of his work at Igalia on [SteamOS](<https://store.steampowered.com/steamos>) and the [Steam Deck](<https://store.steampowered.com/steamdeck>). The name of the scheduler is a bit of a mouthful, but it is focused on making Windows games run better on Linux. SteamOS (and the Steam application for Linux) use [Wine](<https://www.winehq.org/>) and the [Proton](<https://github.com/ValveSoftware/Proton?tab=readme-ov-file#introduction>) compatibility layer. Most of the scheduling decision-making code for LAVD is written in BPF, with a thin, Rust-based user-space piece. 

Some additional information about sched_ext and LAVD can be found in an [article](<https://lwn.net/Articles/991205/>) from last year's LPC. In addition, there were sessions on LAVD as part of the [sched_ext microconference](<https://lpc.events/event/19/sessions/229/#20251212>) this year, including one about [adapting LAVD to be the default scheduler](<https://lpc.events/event/19/contributions/2099/>) for Meta's production fleet. 

Most of his experience that has shaped LAVD is based on Windows games and their needs on Linux. The session was meant to raise awareness of some missing pieces of tooling that could be used to help better support Linux gaming. He hoped to start some discussion of ways to gather better performance information from games and to reliably benchmark them to improve scheduling. 

#### Gaming workloads

There are two things to target in a scheduler for gaming, he said, performance and energy consumption. Gaming is unique because it has ""a widely accepted, universal performance metric, which is frames-per-second"" (or FPS). There are two types of FPS measures: average and the "low 1%" or minimum FPS. The average is a measure of application throughput, while the low 1% measures 99th percentile latency, which is what causes stuttering. So the goal is to provide high average FPS while keeping the low 1% as high as possible so that the gamer does not get angry—it is the most important aspect for the gaming experience. 

[ ![\[Changwoo Min\]](https://static.lwn.net/images/2026/lpc-min-sm.png) ](<https://lwn.net/Articles/1052995/>)

Many gaming devices, such as the Steam Deck, are powered by batteries, so energy efficiency is important. The underlying hardware often uses processors that have a mix of different types of cores (big, medium, and little) with different energy-use characteristics. For games, there are usually multiple tasks, but some of them can be run on the slower, more energy-efficient cores. Most of the tasks are fairly short running, around 1ms, though there are a few that run for 2-3ms or more. There are also tasks that run for less than 100µs, which means there is more flexibility in terms of their task placement, he said. 

In his view, the scheduler can make three separate decisions that affect performance. It chooses which task runs first; if latency-critical tasks are delayed, it can lead to a ""cascading delay or the UI jank or the stuttering"", which affects the user experience. The scheduler also determines how long a task should run; the time slice chosen needs to balance between long enough to benefit from a warm cache, while not so long that tasks monopolize the processor. Beyond that, the scheduler chooses which CPU to use for each task; the cost of making that decision also affects latency and energy efficiency. Quick decisions reduce scheduler overhead, but may increase the other costs because of a bad decision that is made; a problem can also occur if the scheduler overhead gets too high because it is trying to make a better decision. 

Understanding gaming workloads is critical to developing scheduling policies that are optimized for games. When he started out, Min was not confident that he could come up with a gaming-optimized scheduler, so he set out to measure the workloads to see if there were some characteristics that were shared, which the scheduler could use. So he developed [VaporMark](<https://github.com/Igalia/vapormark>)—the name refers to Steam—which analyzes data collected using "`perf sched record`". ""After collecting huge amounts of data, post-processing it for an hour or two, it shows some reports."" 

> [ ![\[VaporMark slide\]](https://static.lwn.net/images/2026/lpc-vapormark-sm.png) ](<https://lwn.net/Articles/1052993/>)

He showed a sample VaporMark report (seen above from his [slides](<https://lpc.events/event/19/contributions/2150/attachments/1951/4162/Steps_Towards_a_Gaming_Optimized_Schedule-lpc2025.pdf>)). It is a useful report, he said, that shows information like the typical run-time duration of each task and can allow determining whether statistically valid predictions can be made about upcoming durations. It provides insight into why and how tasks are being scheduled for the game. 

The key finding that came out of his analysis is perhaps somewhat obvious: a single high-level action, such as moving a character on-screen and emitting a sound based on a key-press event, requires that many tasks work together. Some of the tasks are threads in the game process, but others are not because they are in the game engine, kernel, and device drivers; there are often 20 or 30 tasks in a chain that all need to collaborate. Finding tasks with a high waker or wakee frequency and prioritizing them is the basis of the LAVD scheduling policy. 

#### More tools

VaporMark was useful for understanding the high-level properties of the game, but once he finished a prototype of LAVD, he found that it was inadequate for further work. He tried [Perfetto](<https://perfetto.dev/>), which was better, especially for doing ""microscopic analysis to find out the pathological scheduling behaviors"". It did not solve his analysis problems, though, perhaps because he is not a Perfetto expert, Min said. 

Part of the problem for both VaporMark and Perfetto is that the problems he wants to analyze occur rarely. For example, a game may run smoothly most of the time, but have a latency spike every five minutes or so that results in a drop in the FPS rate. ""How should I catch this?"" He was not able to find a way to trigger on the problem so that his trace showed what led to it; instead he used brute-force methods of sampling every ten seconds, hoping to catch the problem occurring. ""The correlation between the high-level view and the microscopic view is just something missing"" in the tool set. 

Track co-organizer David Vernet noted that Perfetto does have ways to track more than just CPU time, such as memory use, but that it is sampling-based, so it may not provide all of the details that Min is looking for. Another attendee wondered about tracing the latency of key-press events, which is important to certain types of games, especially those favored by professional gamers. That analysis may require more of a view into the game engine than is available, but he wondered if LAVD had done anything to minimize that latency. Min said that LAVD did not focus on that, but that he thought that its policies would naturally prioritize the tasks involved in key-press handling. 

Vernet noted that some games, such as Civilization VI, have CPU-intensive tasks that may need to be handled differently. Some of the workloads where he works (Meta) have similar characteristics and he wondered if Min had done any analysis of the whole chain of tasks from an initial wakeup to the last task completing its work and then using that to derive deadlines and frame rates. Min said that he had not, but that it was an area he wanted to look into; for example, there may be ways to use information from the GPU to help make better CPU-scheduling decisions. 

An attendee raised the issue of choosing which CPU to run a wakee on and whether it makes sense to move tasks to other CPUs based on their relationships. Min said that making those decisions is ""a pretty tricky thing"". He has been looking at the timeline-management aspect of the scheduler, but there are other resources to consider, such as cache locality and memory distance. There is a need for a tool to generate a ""more holistic view of the resource contention""; it might be possible to do that with Perfetto, but it still remains to be done. 

#### And benchmarks

Another problem he has encountered while working on the scheduler is that benchmarking games is ""super hard"". In order to show that a new scheduling policy is better (or worse) than an old one, a ""reliably reproducible benchmark is absolutely necessary"", but games lack that, unlike other domains, such as databases, that have standard benchmarks. Fortunately, some games, such as Cyberpunk 2077, Far Cry, and Forza Horizon, have an in-game benchmark, which is useful; others, like Counter-Strike, allow replaying recorded game sequences, which also helps. 

Unfortunately, it is not clear how representative the results from those games are with respect to other games that lack those abilities; for example, some users have complained that LAVD does not work well with the games using the Unreal engine. Another problem is that games are updated frequently so any data gathered can quickly age out; comparing results before and after a 2GB game update ""does not make any sense"", he said. The in-game benchmarks sometimes rely on the performance of external resources, such as the game server, as well. 

Even when reliable benchmark results are available, it is difficult to correlate observed FPS changes with the scheduler change(s) that caused them. So, microbenchmarks are of interest as well, but most of the ones he has found for schedulers (e.g. [stress-ng](<https://github.com/ColinIanKing/stress-ng?tab=readme-ov-file#stress-ng-stress-next-generation>) and [hackbench](<https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/hackbench>)) are focused on stressing schedulers and measuring the scheduling overhead. That is useful, but improving those numbers does not always lead to better game performance. 

There are some microbenchmarks that mimic specific workloads; one example is [schbench](<https://kernel.googlesource.com/pub/scm/linux/kernel/git/mason/schbench>), which is meant to reproduce the scheduling characteristics of production web-server workloads. There is a need for something like that to mimic gaming workloads, he said. Vernet asked how that kind of benchmark might be built; would it use existing game engines or somehow incorporate [AAA games](<https://en.wikipedia.org/wiki/AAA_\(video_game_industry\)>)? Min said that he was not sure how to approach it; one problem is that the most important games and engines to consider changes based on market share. 

One thing that could help is a suite of benchmarks. He noted that [CloudSuite](<https://github.com/parsa-epfl/cloudsuite?tab=readme-ov-file#cloudsuite-40>) provides a set of representative cloud applications packaged into a benchmark suite. A collection of games could perhaps be turned into a "gamesuite" for scheduler testing; it would augment a few microbenchmarks that run for a short time to give quick feedback. The in-game benchmarks often only give the average FPS, ""which is useful but not ideal""; they lack the low 1% and minimum FPS information that is also really needed. 

André Almeida, who was the other track organizer, suggested using [MangoHud](<https://github.com/flightlessmango/MangoHud?tab=readme-ov-file#mangohud>) to gather more FPS data; Min concurred, but said that MangoHud needs to be carefully configured. When using it in "batch" mode, the logging from MangoHud can interfere with the game, which can cause stuttering. A gamesuite could potentially have a recommended MangoHud configuration along with a set of representative games to run. 

#### Locks

His focus has largely been on Windows games, which use the synchronization primitives of that underlying operating system. For Linux, those are emulated by the Wine layer using futexes, but there is still work to be done. Most of the tasks in a Windows game are short-running and computation-intensive, so it is important to avoid situations where locking prevents a task from quickly completing. That means avoiding lock-holder preemption (or priority inversion)—when a lock holder is scheduled out, but a newly scheduled task needs the lock so it cannot progress. The work that Almeida has been doing to [rework the futex API](<https://lwn.net/Articles/823513/>) will also help with priority inversion, Min said. 

In the kernel, [proxy execution](<https://lwn.net/Articles/934114/>) seems to him to be the right approach, but it does not help with user-space futexes. LAVD attempts to track tasks and futexes to determine when to provide a longer time slice to a task in order to give it time to release a lock; that is not a complete solution to the problem, however. Meanwhile, Wine is moving to using the [ntsync driver](<https://docs.kernel.org/userspace-api/ntsync.html>), which is designed to better support the Windows NT synchronization primitives. Eventually, ntsync and proxy execution should help alleviate these kinds of problems, he thought. 

Almeida noted that, on the fast path, the kernel does not know which task holds a futex, so it does not have that information for making scheduling decisions. He has heard that FreeBSD programs always inform the kernel about the futex owner so that it can avoid priority inversion. The cost of a system call is high, though, so requiring that could be problematic. There was some discussion of other Windows lock types and whether those were supported well in Linux; the upshot seemed to be that those are built on top of the futex interface, so fixing futexes should take of care of most problems for Windows locks—at least for games. 

An attendee asked whether LAVD used the same scheduler for all games on the Steam Deck. Min said that LAVD is a single scheduler, though there are some tuning knobs that can be changed with command-line parameters. The attendee wondered if it made sense to consider something like ""profile-guided scheduling"" where a game's profile could be used to set parameters for the scheduler. For the most part, the profiles would only need to be engine-specific, since most games use an off-the-shelf or internal engine that governs their performance characteristics. Min agreed that it sounded like an interesting approach, but it was not one he has explored. 

In a followup question, he wondered about reproducible game testing; if there is no network server involved, are there elements beyond the generation of random numbers that need to be constrained in order to reproduce a game run? Min said that random numbers were the main impediment to determinism for a game, but there are some other factors at play too. If a game runs too fast, it may cause the CPU to overheat and be throttled; that will obviously change the characteristics as well. 

The session ended with a bit of discussion about using the `WF_SYNC` (or `WF_CURRENT_CPU`) flag to indicate tasks that will return to sleep immediately after waking. That can help the kernel make better scheduling decisions and is used on Android that way. Interested readers may want to consult the [YouTube video](<https://www.youtube.com/watch?v=5F-vQgv4sI0>) of the session for the finer details. 

[ I would like to thank our travel sponsor, the Linux Foundation, for assistance with my travel to Tokyo for Linux Plumbers Conference. ]
