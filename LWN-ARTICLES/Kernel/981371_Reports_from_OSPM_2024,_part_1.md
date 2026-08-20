---
title: "Reports from OSPM 2024, part 1"
url: https://lwn.net/Articles/981371/
date: "July 11, 2024"
category: "Latency; Scheduler-EEVDF"
author: "July 11, 2024 This article was contributed by Giovanni Gherdovich OSPM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 11, 2024

This article was contributed by Giovanni Gherdovich

* * *

[OSPM](<https://lwn.net/Archives/ConferenceIndex/#OS-Directed_Power-Management_Summit-2024>)

The sixth edition of the [Power Management and Scheduling in the Linux Kernel (OSPM) Summit](<https://retis.sssup.it/ospm-summit/>) took place on May 30-31 2024, and was graciously hosted by the [Institut de Recherche en Informatique de Toulouse (IRIT)](<https://www.irit.fr/en/home>) in Toulouse, France. This is the first of a series of articles describing the discussions held at OSPM 2024; topics covered include latency hints, energy-aware scheduling, ChromeOS, and user-space schedulers. 

IRIT is one of the largest French joint research units (unité mixte de recherche), with more than 700 collaborators and the stated mission of placing ""mankind and its environment at the heart of computer science""; it is a fitting partner for a gathering of kernel developers looking to optimize energy consumption in Linux. The list of sponsors for the Summit doesn't end at the IRIT: it also includes [Arm](<https://arm.com/>), [Linaro](<https://linaro.org/>), and the [Scuola Superiore Sant'Anna](<https://www.santannapisa.it/en>) in Pisa as the host for the Summit web site. 

The following contains summaries of the sessions from the first half day of the event. A recording of the entire summit is available as a playlist on the [IRIT YouTube channel](<https://www.youtube.com/playlist?list=PLXJaBJgIrHE3g1hdEpypbFR4SHLEX6NfN>). 

#### How to set some latency hints to CFS tasks

Vincent Guittot started out the summit with an attempt at characterizing the various latency constraints that tasks may have, as a way of directing a renewed effort toward [a "latency-nice" interface](<https://lwn.net/Articles/887842/>). Since the algorithm for the scheduler has changed from completely fair scheduling (CFS) to [EEVDF](<https://lwn.net/Articles/925371/>), the prior discussions don't apply anymore. He sees two main groups of tasks: those that are sensitive to scheduling latency in one way or another, and those that aren't, but which benefit from a longer time slice and little to no preemption.

Guittot raised the issue that, with EEVDF, a task's virtual runtime doesn't account for performance variations caused by DVFS (CPU-frequency scaling). When executing on a CPU running at a lower frequency, the task makes less progress, yet it consumes the same amount of absolute time as another that's running at a higher clock. He thinks that the virtual runtime can be easily normalized and made frequency-invariant, but he's also concerned about how this may affect the limit on lag (a measure of under-service experienced by a task) offered by the original EEVDF algorithm.

With [the latest EEVDF patches](<https://lwn.net/Articles/969062/>), user-space applications can set their own time slice using the [`sched_setattr()`](<https://man7.org/linux/man-pages/man2/sched_setattr.2.html>) system call. This puts the burden of measuring their own run time and requesting a suitable time-slice value onto applications. When considering an interface that allows a tighter collaboration between kernel and applications to set an appropriate slice value, Daniel Bristot de Oliveira observed that there isn't, as of now, a means for user space to notify the scheduler of the end of a work cycle, which would be beneficial for those tasks with a somewhat regular and recurrent nature. The [`sched_yield()`](<https://man7.org/linux/man-pages/man2/sched_yield.2.html>) system call comes close, but Peter Zijlstra would prefer adding a new system call for that specific purpose.

Enrico Bini asked if it would be advantageous to group tasks in cores according to their time granularity (time-slice length). The rationale is that, if the scheduler mixes tasks with long and short slices, it seems inevitable that either the long-running tasks will often be preempted, or the short ones will experience high latency. Zijlstra agreed it would be beneficial to incorporate this logic into the load balancer.

Dietmar Eggemann stressed that, whatever the latency API will turn out to be, it is necessary to identify an agreed-upon set of benchmarks to evaluate progress along the way. He mentioned that, in Android, many of the ideas proposed during this session have been tested, and some turned out to make no difference.

For the recording of Guittot's session on latency hints, see [this video](<https://www.youtube.com/watch?v=BI3QvJVa0IM>). 

#### Fixing, improving, and extending energy-aware scheduling

During the second session of the morning, Guittot identified a set of quirks in the [energy-aware scheduling (EAS)](<https://lwn.net/Articles/749900/>) engine that he's determined to correct. He presented data from an Arm "three-gear" big.LITTLE CPU. The device has four "little" (slow but energy-efficient) cores, three "mid" cores, and one "big" (fast but energy-expensive) core. A number of concerns came to light: 

  * **A small task runs on a big core for no reason.** Consider a small, periodic task that needs to run quickly with more CPU time than a little core could provide. A mid core would be perfect, but the big core is too much. It is possible to use [utilization clamping](<https://lwn.net/Articles/762043/>) and set a minimum amount of CPU power for this task to keep it off the little cores. If a larger task is occupying one of the mid cores, though, EAS would place the smaller one on the big core, even if there's a free mid. Guittot's proposed solution is to apply the cost of the predicted operating performance point (OPP) only for the runnable time of tasks, as reported by [PELT](<https://lwn.net/Articles/531853/>) historical data. 
  * **Spare CPU capacity vs. actual OPP cost.** At task wakeup, EAS picks a candidate core in each performance domain (little, mid, and big), according to each core's load compared to its capacity; Guittot argues that looking directly at the cost for the predicted OPP would yield better placements. Energy-model maintainer Lukasz Luba isn't fully convinced, as there are diminishing returns when using more sophisticated placement algorithms. 
  * **More room than spare capacity would suggest.** When cores report having no spare capacity, EAS isn't effective and the resulting placements can be highly imbalanced. In Guittot's example, clamping the maximum utilization exacerbates this condition, as cores look smaller than they are. Guittot pointed out that when a task has a utilization that is larger than the capacity of a CPU, it can be serviced nonetheless; it only needs to run a little longer. 
  * **Over-utilization scrambles all placements.** Imagine some kernel thread waking up and running for just a handful of microseconds. If that causes a core to cross the 80% utilization tipping point, EAS will be disengaged globally and the ordinary load balancer will take over, invalidating all previous efforts to optimize the system's power consumption. Guittot said that, for those cores and domains that aren't overutilized, it is better to continue using an energy-efficient task placement algorithm. 
  * **`uclamp_max` prevents EAS from running often enough.** Smartphones are becoming increasingly powerful, but all that horsepower isn't meant to be exercised constantly. The practice is thus to clamp the maximum utilization of tasks, but as Guittot reports, this results in less-frequent wakeups, and thus fewer opportunities for EAS to optimize for power. Guittot intends to introduce the notion of misfit tasks with respect to power needs, and when such cases are detected, use load balancing to pull these tasks onto a more appropriate core. 

For the recording of Guittot's session on EAS, see [this video](<https://www.youtube.com/watch?v=PHEBAyxeM_M>). 

#### ChromeOS and EEVDF

The most important application in a ChromeOS laptop is, of course, the Chrome web browser; Youssef Esmat described the progress he and his team have made in integrating the EEVDF scheduler with these systems, especially with the browser. Chrome's threads can be classified by their tolerance to latency in three groups: user-interface-related threads, which capture user input and need the highest priority, background threads, such as those representing inactive tabs, which are not critical to user experience, and everything else, sitting somewhere in between these two extremes.

Esmat stressed that Chrome's perceived performance depends heavily on satisfying the latency and throughput needs of its UI threads. These have to be serviced promptly, and at the same time are vulnerable to preemption as they are long-running. The EEVDF algorithm is based on the assumption that tasks, in general, don't behave like that. Instead, it assumes that, if a task needs to be scheduled quickly, it also completes quickly and, conversely, if it runs for a prolonged time, it can tolerate higher scheduling latencies.

For this analysis, he compared the EEVDF scheduler with CFS, using web-application workloads such as Google Meet and Google Docs, and measuring input latency and percentage of dropped frames. Stock EEVDF performed worse than CFS; Esmat discovered that quadrupling the base EEVDF time-slice length, which defaults to 1.5ms for a two-core machine, restored parity with CFS. The real gain came from removing the eligibility mechanism altogether from EEVDF and scheduling only based on the deadline value: this change would make the new scheduler outperform CFS by some 30% in the metrics of interest. The reason for this improvement is that, with eligibility, background threads would become eligible and preempt UI threads before they could finish their time slice; as discussed, the quality of Chrome's user experience depends on UI threads running until completion.

Being the clear winner in the initial comparison, the version of EEVDF with a large base time slice and no eligibility was selected for field testing, and deployed to users for real-world comparison with CFS. The new scheduler outperformed CFS in page-load speed (time to first content paint, time to largest content paint) and almost all aspects of UI input latency: key pressed, mouse pressed and touch pressed. There is one regression though, in latency for gesture scrolling, and Esmat's team is working on a reproducer so that the issue can be studied and solved.

For the recording of Esmat's session on EEVDF benchmarking, see [this video](<https://www.youtube.com/watch?v=dxsbtjJRSm4>). 

#### Writing a Linux scheduler in Rust that runs in user space

Andrea Righi introduced [scx_rustland](<https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_rustland
>), a framework to write CPU schedulers for Linux that run as user-space programs. Initially a project aimed at teaching operating systems concepts to undergraduate students, it led Righi to appreciate the convenience of the change/build/run workflow to modify the kernel's behavior without rebooting. This effort was able to prove that user-space scheduling is not only possible, but can even reach respectable performance, such as video gaming at 60 frames per second while compiling the Linux kernel at the same time. A lively discussion followed, centered on the merits and disadvantages of Righi's and similar frameworks, and on whether or not there's a place in Linux for something like [sched_ext](<https://lwn.net/Articles/974387/>), the kernel patch upon which Righi's work is based. 

Juri Lelli recalled starting out in scheduler development; after trying both a research framework for scheduler plugins and coding his algorithms directly in the kernel, he concluded the learning curve was the same. Righi replied that creating schedulers in user space, in Rust, is of great appeal for the new generation. Bristot objected that a plugin framework only relieves developers from some implementation boilerplate, which is definitely not the hard part of scheduler development. Esmat argued that, for undergraduate students, there's a lot of value in quickly applying their changes to a running Linux system; it could get them excited about kernel development. Himadri Chhaya-Shailesh believes sched_ext will be a great teaching aid at universities, since one can test scheduler code without recompiling the kernel. In her experience as a lecturer, kernel compilation takes a large part of the lab sessions.

Thomas Gleixner maintains that the core machinery for scheduling in Linux isn't modular enough to expose hooks such as those added by sched_ext, at least not without introducing large technical debt. John Stultz asked what sched_ext is offering in terms of correctness testing for user-provided schedulers; Righi mentioned the sched_ext watchdog, which prevents scheduling starvation at run time. I agreed that a generic test suite for custom schedulers would be helpful to distribution vendors; it would facilitate the interaction during support requests, allowing users to show that their third-party scheduler meets basic quality standards. Christian Loehle commented on benchmark results achieved with custom schedulers: getting, say, higher frames per second than stock Linux in gaming, without any mention of power consumption, for example, is an unfair comparison. Righi replied that custom schedulers can afford to be workload-specific; reporting only the metric the scheduler set out to optimize seems acceptable in this context.

For the recording of Righi's session on user-space scheduling, see [this video](<https://www.youtube.com/watch?v=HQRHo8E_4Ks>). 

[Thanks to Daniel Lezcano and Juri Lelli for proofreading this article.]
