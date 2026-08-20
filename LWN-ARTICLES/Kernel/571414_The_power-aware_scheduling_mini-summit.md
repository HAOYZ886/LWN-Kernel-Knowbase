---
title: "The power-aware scheduling mini-summit"
url: https://lwn.net/Articles/571414/
date: "October 23, 2013"
category: "Power management-CPU scheduling; Scheduler-and power management"
author: "By Jonathan Corbet October 23, 2013 Kernel
Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 23, 2013

* * *

[Kernel Summit](<https://lwn.net/Archives/ConferenceByYear/#2013-Kernel_Summit>)

The first day of the 2013 Kernel Summit was set aside for mini-summits on various topics. One of those topics was the controversial area of power-aware scheduling. Numerous patches attempting to improve the scheduler have been posted, but none have come near the mainline. This gathering brought together developers from the embedded world and the core scheduler developers to try to make some forward progress in this area; whether they succeeded remains to be seen, but some tentative forward steps were identified. 

[![\[Morten Rasmussen\]](https://static.lwn.net/images/conf/2013/lce-ks/MortenRasmussen-sm.jpg)](<https://lwn.net/Articles/571504/>) Morten Rasmussen, the author of the [big LITTLE MP patch set](<https://lwn.net/Articles/541005/>), was one of the organizers of this session. He started with a set of agenda items and supporting slides; the topics to be discussed were: 

  * Unification of power-management policies, which are currently split among multiple, uncoordinated subsystems. 

  * Task packing. Various patch sets have been posted, but none have really gotten anywhere. 

  * Power drivers and, in particular, the creation of proper abstractions for hardware-level power management. 

He attempted to get started with a discussion of the cpufreq and cpuidle subsystems, but didn't get very far before the conversation shifted. 

#### The need for metrics

[![\[Paul McKenney and Ingo Molnar\]](https://static.lwn.net/images/conf/2013/lce-ks/Paul+Ingo-sm.jpg)](<https://lwn.net/Articles/571505/>) Ingo Molnar came in with a complaint: none of the power-management work starts with measurements of the system's power behavior. Without a coherent approach to measuring the effects of a patch, there is no real way to judge these patches to decide which ones should go in. We cannot, he said, merge scheduler patches on faith, hoping that they somehow make things better. 

What followed was a long and wide-ranging discussion of how such metrics might be made. What the scheduler developers would like is reasonably clear: how long did it take to execute a defined amount of work, and how much energy was consumed in the process? There is also some interest in what worst-case latencies were experienced by the workload while it was running. With reproducible numbers for these quantities, it should be possible to determine which patches help with which workloads and make some progress in this area. 

Agreement with this approach was not universal, though. Mark Gross of Intel made the claim that this kind of performance metric was "the road to hell." Instead, he said, power benchmarks should focus on low-level behavior like processor sleep ("C") states. For any given sleep state, the processor must remain in that state for a given period of time before going into that state actually saves power. So the cpuidle subsystem must make an estimate of how long the processor must be idle before selecting an appropriate sleep state. The actual idle period is something that can then be measured; over time, one can come up with a picture of how well the kernel's estimates of idle time match up with reality. That, Mark said, is the kind of benchmark the kernel developers should be using. 

Others argued that idle-time estimation accuracy is a low-level measurement that may not map well onto what the users actually want: their work done quickly, without unacceptable latencies, and with a minimal use of power. But actual power-usage measurements have been hard to come by; the processor vendors appear to be reluctant to expose that information to the kernel. So the group never did come up with a good set of metrics to use for the evaluation of scheduler patches. In the end, Ingo said, the first patch that adds a reasonable power-oriented benchmark to the `tools` directory will go in; it can be refined from there. 

#### What to do

From there, the discussion shifted toward how the scheduler's power behavior might be improved. It was agreed that there is a need for a better mechanism by which an application can indicate its latency requirements to the kernel. These latency requirements then need to be handled carefully: it will not do to keep all CPUs in the system awake because an application running on one of them needs bounded latencies. 

There was some talk of adding some sort of energy use model to a simulator — either [linsched](<https://lwn.net/Articles/487701/>) or the `perf sched` utility — to allow for standardized testing of patches with specific workloads. Linsched was under development by an intern at Google, but the work was not completed, so it's still not ready for upstreaming. Ingo noted that the resources to do this work are available; there are, after all, developers interested in getting power-aware code into the scheduler. 

[![\[Paul Turner and Peter Zijlstra\]](https://static.lwn.net/images/conf/2013/lce-ks/Paul+Peter-sm.jpg)](<https://lwn.net/Articles/571506/>) Rafael Wysocki asked the scheduler developers: what information do you need from the hardware to make better scheduling decisions? Paul Turner responded: the cost of running the system in a given configuration; Peter Zijlstra added that he would like to know the cost of starting up an additional CPU. The response from Mark is that it all depends on which generation of hardware is in use. In general, Intel seems to be reluctant to make that information available, an approach which caused some visible frustration among the scheduler developers. 

Over time the discussion did somewhat converge on what the scheduler community would like to have. Some sort of cost metric should be attached to the scheduling domains infrastructure; it would tell the scheduler what the cost of firing up any given processor would be. That information would have to appear at multiple levels; bringing up the first processor in a different physical package will clearly cost more than adding a processor in an already-active package, for example. 

Tied into this is the concept of performance ("P") states, generally thought of as the "CPU frequency." The frequency concept is somewhat outdated, but it persists in the kernel's cpufreq subsystem which, it was mostly agreed, should go away. The scheduler _does_ need to understand the cost (and change in performance) associated with a P-state change, though; that would allow it to choose between increasing a CPU's P state or moving a task to a new CPU. How this information would be exposed by the CPU remains to be seen, but, if it were available, it would be possible to start making smarter scheduling decisions with it. 

How those decisions would be made remains vague. There was talk of putting together some kind of set of standard workloads, but that seems premature. Paul suggested starting with a set of "stories" describing specific workloads in a human-comprehensible form. Once a collection of stories has been put together, developers can start trying to find the common themes that can be used to come up with algorithms for better, more power-aware scheduling. 

There was some brief discussion of Morten's recent [scheduler patches](<https://lwn.net/Articles/570353/>). It was agreed that they provide a reasonable start for the movement of CPU frequency and idle awareness into the scheduler itself. A focus on moving cpuidle into the scheduler first was suggested; most developers would rather see cpufreq just go away at this point. 

And that was where the group was reminded that lunch had started nearly half an hour before. The guidance produced for the power-aware scheduling developers remains vague at best, but there are still some worthwhile conclusions, with the need for a set of plausible metrics being at the top of the list. That should be enough to enable this work to take a few baby steps forward. 

(For more details, see [the extensive notes from the session](<http://etherpad.osuosl.org/energy-aware-scheduling-ks-2013>) taken by Paul McKenney and Paul Walmsley). 

[Your editor would like to thank the Linux Foundation for supporting his travel to Edinburgh.]
