---
title: Improved load balancing with machine learning
url: https://lwn.net/Articles/1027096/
date: "July 1, 2025"
category: "Scheduler-Extensible scheduler class"
author: "By Jonathan Corbet July 1, 2025 OSSNA"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 1, 2025

* * *

[OSSNA](<https://lwn.net/Archives/ConferenceIndex/#Open_Source_Summit_North_America-2025>)

The [extensible scheduler class](<https://lwn.net/Articles/922405/>) ("sched_ext") allows the loading of a custom CPU scheduler into the kernel as a set of BPF functions; it was merged for the 6.12 kernel release. Since then, sched_ext has enabled a wide range of experimentation with scheduling algorithms. At the 2025 [Open Source Summit North America](<https://events.linuxfoundation.org/open-source-summit-north-america/>), Ching-Chun ("Jim") Huang presented work that has been done to apply (local) machine learning to the problem of scheduling processes on complex systems. 

Huang started with a timeline of Linux scheduler development, beginning with the adoption of the completely fair scheduler (CFS) in 2007. Various efforts were made to write alternatives to CFS for specific use cases, notably the 2009 [submission of BFS](<https://lwn.net/Articles/351499/>), and the 2016 [MuQSS submission](<https://lwn.net/Articles/720227/>), both from Con Kolivas. In 2023, the [EEVDF scheduler](<https://lwn.net/Articles/925371/>) showed up as an enhancement to, and eventual replacement for, CFS. The following year, finally, saw the merging of sched_ext, after some extensive discussion. 

In other words, he said, it took 17 years from the beginning of the CFS era to get to the point where an extensible scheduler was added to Linux. That period reflects a long-held opinion that one scheduler could be optimal for all situations. This position was [clearly expressed by Linus Torvalds](<https://lore.kernel.org/all/alpine.LFD.0.999.0710021410470.3579@woody.linux-foundation.org/>) in 2007: 

> The arguments that "servers" have a different profile than "desktop" is pure and utter garbage, and is perpetuated by people who don't know what they are talking about. The whole notion of "server" and "desktop" scheduling being different is nothing but crap. 

The reality of the situation, Huang said, has changed since then. In 2007, machines typically had a maximum of four CPUs, those CPUs were all equivalent to each other, and the workloads were relatively simple. In 2025, instead, systems can have over 48 cores with heterogeneous CPUs and complex requirements for throughput, latency, and energy consumption. The heuristics used by CFS (and EEVDF) were designed for the simpler times, and are no longer optimal. 

The complexity of modern systems comes in numerous forms. NUMA systems can perform badly if workloads are scheduled far from their memory. The CFS scheduler often makes bad placement choices; as a result, administrators are being forced to pin processes to specific CPUs or to partition their systems to regain performance. Heterogeneous systems have multiple CPU types with different performance and energy-use characteristics, and even different instruction sets. Putting a task on the wrong type of CPU can affect performance, waste energy, or, in some cases, even cause a crash due to an instruction-set mismatch. 

The types of workloads being seen now add complications of their own. A gaming workload, for example, often features a combination of rendering and streaming tasks. The rendering is latency-sensitive and should run on high-performance cores, while the streaming can run on the more efficient cores. A scheduler that treats both task types equally will end up causing dropped frames. The sort of network processing involved with 5G networking involves a combination of tight latency constraints and CPU-intensive work. Even a modern development environment involves challenges, with a combination of CPU-intensive tasks (compilation, for example) and interactive tasks. Bad scheduler decisions can lead to lots of context switches and an unresponsive user interface. 

The end result of all this, Huang said, is that any scheduler using a single, fixed algorithm is fundamentally broken. All of the traditional schedulers do exactly that. They have brought a simple system view into a world where a typical computer has billions of possible states, and their limitations are showing. 

The sched_ext framework offers a potential solution, an environment where schedulers can evolve to meet contemporary challenges. Huang took as a case study the [Free5GC project](<https://free5gc.org>), which is creating an open-source solution for 5G network processing. Its data-plane processing, in particular, is subject to a number of difficult constraints. It has a number of CPU-bound tasks, but also has some strict latency constraints. The CPU scheduler must be able to balance these constraints; CFS often fails to do so optimally. 

The project [experimented](<https://free5gc.org/blog/20250509/20250509/>) with a sched_ext scheduler called "scx_packet". It used a relatively simple algorithm: half the CPUs in the system were reserved for latency-sensitive network-processing tasks, while the other half were given over to CPU-bound general processing. But this scheduler treated all network traffic equally — voice calls, web browsing, and streaming all went to the same CPUs. That could cause voice data to be blocked behind download traffic, and emergency calls had the same priority as social-media activity. This approach also led to some CPUs being overloaded, while others were idle, as the workload shifted. Finally, some packets require much more processing than others; the processing of the more CPU-intensive packets should be scheduled separately. 

This experience led the Free5GC developers to look into machine learning. Scheduling on such systems has many dimensions of input to consider; it is, he said, ""the perfect problem domain"" for machine learning. Among other things, the scheduler must consider the priority of each task, its CPU requirements, its virtual run time so far, and its recent CPU-usage patterns. The load on each CPU must be taken into account, as must NUMA distance, cache sharing, and operating frequency. Then, of course, there are the workload-specific factors. 

A new sched_ext scheduler (based on [scx_rusty](<https://sched-ext.com/docs/scheds/rust/scx_rusty>)) was developed to try to take all of these parameters into account and decide when a task should be moved from one CPU to another. It initially runs in a data-collection mode, looking at migration decisions and the results from them; these decisions are then used to train a model (in user space) that is subsequently stored in a BPF map. The scheduler can then use this model inside the kernel to make load-balancing decisions. The outcome of these decisions is continually measured and reported back to user space, which updates the model over time. 

Implementing this scheduler required overcoming an obstacle unique to the kernel environment. Neural-network processing involves a fair amount of floating-point arithmetic, but use of floating-point instructions is not allowed in kernel code (saving the floating-point-unit state on entry to the kernel would have a heavy performance cost, so the kernel does not do that). A form of fixed-point arithmetic was adopted for the neural-network processing instead. 

In a test using the all-important kernel-compilation benchmark, this scheduler produced a 10% improvement in compilation time over the EEVDF scheduler. The number of task migrations was reduced by 77%. 

Huang concluded with a summary of why machine learning works in this context. Scheduling in this complex environment is, he said, a pattern-recognition problem, and neural networks are good at that task. The scheduler is able to balance competing goals and automatically re-trains itself for new architectures and workloads. The scheduler is able to take 15 separate parameters into account for each migration decision, and to adjust its model based on the results. 

The [slides from Huang's talk](<https://static.sched.com/hosted_files/ossna2025/d2/Improve-Load-Balancing-With-Machine-Learning-Techniques-based-on-sched_ext.pdf>) are available for interested readers. The source for the machine-learning-based sched_ext scheduler can be [found on GitHub](<https://github.com/vax-r/scx/tree/scx_rusty_MLLB>). 

[Thanks to the Linux Foundation for supporting my travel to this event.]
