---
title: Controlling memory management with BPF
url: https://lwn.net/Articles/1072538/
date: "May 15, 2026"
category: "BPF-Memory management; Control groups-Memory controller"
author: "By Jonathan Corbet May 15, 2026 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 15, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Roman Gushchin began his session in the memory-management track of the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) by saying that the community has seen a lot of proposals adding BPF-based interfaces for memory management. None of them have made their way into the mainline, though. He wanted to explore the ways in which BPF might be helpful and the obstacles that have kept BPF-based solutions out so far. This session was followed by a discussion led by Shakeel Butt on what the requirements for a new, BPF-based interface for memory control groups might look like. 

#### Obstacles to BPF integration

Existing efforts have tried to capture a number of different memory-management heuristics, he began. There have been proposals to use BPF to control [out-of-memory handling](<https://lwn.net/Articles/1019230/>), [NUMA balancing](<https://lwn.net/ml/all/20260113121238.11300-1-laoar.shao@gmail.com/>), [memory control groups](<https://lwn.net/ml/all/cover.1770194182.git.zhuhui@kylinos.cn/>), [page-cache eviction](<https://lwn.net/ml/all/0b1293ca-7a1f-4358-bc20-15784452238d@columbia.edu/>), and more. There are more interesting ideas that have not yet been pursued, including readahead control, [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>), [kernel samepage merging](<https://lwn.net/Articles/953141/>), and guest-memory control. Readahead, in particular, is a messy set of heuristics, but it is important for performance. 

[![\[Roman Gushchin\]](https://static.lwn.net/images/conf/2026/lsfmm/RomanGushchin-sm.png)](<https://lwn.net/Articles/1072542/>) There are a number of obstacles to the addition of BPF interfaces for the memory-management subsystem, he said; he would cover them from the least important to the most. The first was concerns about out-of-tree BPF programs. Kernel developers want to see production-quality code land in the mainline, but that is not how BPF is working now. There are production-quality [sched_ext schedulers](<https://lwn.net/Articles/974387/>), for example, but they are all stubbornly out of tree. BPF maintainer Alexei Starovoitov said that ""sched_ext was a mistake"", in that it did not bring any production schedulers with it into the mainline. That is a hard situation to fix now, he said. It would be good to have a good, in-tree out-of-memory handler; if nothing else, it would help developers to judge the proposed interfaces. 

Including BPF programs in the kernel tree does not seem to be controversial, Gushchin said, so the real question is how far developers should go. A first step would be to just include the source for people to examine and play with. Automatic loading of included BPF programs could be a good second step, Starovoitov said; it would let people use the included BPF programs easily. Gushchin suggested that a BPF implementation of [systemd-oomd](<https://www.freedesktop.org/software/systemd/man/latest/systemd-oomd.service.html>) would provide a good example of how that subsystem works. 

Another obstacle is the current inability to attach [struct ops](<https://lwn.net/Articles/811631/>) programs to control groups. BPF _programs_ can be attached, but not those using the struct ops interface. He has an implementation for the out-of-memory handler, but sched_ext uses a different solution. 

Then, there is the issue of safety and fallback; a broken BPF memory-management program could easily make the system unusable. This is the hardest issue to solve, at least from Gushchin's perspective; it is hard even to define what "safety" means in this context. Time-based fallbacks are hard to implement and ugly, he said. Memory-management actions can be wrapped into monitored kfuncs, but that leads to non-generic solutions that can hurt performance. The acceptable level of service needs to be defined; a traffic-control program that drops all packets is OK, but a sched_ext scheduler that starves half of the tasks in the system is less so. What should happen if a faulty BPF program is loaded and the system can no longer reclaim memory? 

There will always be concerns about performance in hot paths, which will make it hard to justify adding BPF programs in the hottest of them. The memory-management subsystem depends heavily on batching for performance, raising the question of whether BPF programs should run before or after batching is done. He suggested that batching should happen first, but that makes it impossible to control the batching itself with a BPF program. 

Finally, the most important obstacle, he said, is ABI stability; this concern had been most recently [raised by David Hildenbrand](<https://lwn.net/ml/all/014f3c0a-7c6f-4f64-95cd-b7b69d804880@kernel.org/>) on the mailing list. In person, Hildenbrand said that there was some confusion about what providing hooks for BPF programs means; are they a permanent memory-management feature? The community may not want to commit to keeping those hooks around indefinitely. That concern has led to a decision not to provide hooks for the management of transparent huge pages; nobody knows what the picture will look like in five years, he said, so it will not be possible to get the interface right. 

What will happen, he said, is that memory-management developers will wake up someday and realize that some aspect of the interface should be done differently. If they act on that realization, programs will break and people will get angry. Perhaps the solution is to only commit to supporting BPF programs that are maintained in the kernel tree itself. Hildenbrand concluded by saying that he sees the value of using BPF, but worries that adding interfaces may commit the subsystem to maintaining features that it regrets in the future. 

The session was out of time at this point. At the conclusion, Gushchin said that it would be important to add only the most generic sorts of BPF hooks. So, for example, a hook to assign out-of-memory scores would be a bad idea, since a future out-of-memory killer might not use them. But a hook to free some memory under pressure, perhaps by killing some processes, could be useful. 

#### Reimagining memory control groups

[![\[Shakeel Butt\]](https://static.lwn.net/images/conf/2026/lsfmm/ShakeelButt-sm.png)](<https://lwn.net/Articles/1072550/>) Butt followed immediately after with a discussion of how he might like to see the kernel's [memory controller](<https://docs.kernel.org/admin-guide/cgroup-v1/memory.html>) evolve, and how BPF might fit into that. He started by saying that the memory controller distributes memory resources hierarchically, implementing both hard and soft limits. Any given group will be allowed to use up to its hard limit when memory is plentiful, but will be squeezed back to the soft limit when memory is tight. 

The memory controller has a number of challenges, he said. The enforcement of limits is inflexible and disruptive; since it happens synchronously, it can cause unexpected stalls in latency-sensitive threads. Its interfaces have proved hard to evolve, since significant changes will break the existing ABI, which kernel developers are not allowed to do. It would be nice, he said, to have a mechanism that would make it possible to experiment with alternatives. 

The goal of a new interface would be to provide capabilities to support a wide variety of use cases. One example use case was provided in [his session proposal](<https://lwn.net/ml/linux-mm/20260307182424.2889780-1-shakeel.butt@linux.dev/>) and repeated during the session itself: 

> Policy: "keep system-level memory utilization below 95 percent; avoid priority inversions by not throttling allocators holding locks; trim each workload's usage to its working set without regressing its relevant performance metrics; collaborate with workloads on load shedding and memory trimming decisions; and under extreme memory pressure, collaborate with the OOM killer and the central job scheduler to kill and clean up a workload." 

A new memory controller, he said, would need to provide memory-use notifications that applications can act on. It needs to support background reclaim, so that memory limits can be enforced without stalling running threads. Memory-use throttling should be aware of threads that hold locks to avoid priority-inversion problems. User space needs to be able to influence throttling in other ways, with the ability to, for example, identify specific threads that should be exempt to an extent. The controller should also support memory tiering, providing control over how pages are moved between tiers. 

There was not much time to go into how this new interface would work; this effort, in general, appears to be in an early stage. Butt said that a new BPF callback, `bpf_memcg_charge_succeed()`, could be added to inform a BPF program about an increase in memory usage; that program might then respond by initiating background reclaim. Other callbacks could inform the program when a control group has reached a usage watermark (or hit a usage limit), relying on the program to provide hints on how to respond. That program might initiate some sort of reclaim, but it could also inform the application of the situation with the expectation that said application would respond by reducing its memory usage. 

At the end, a member of the audience said that a useful feature would be the ability to introspect what types of memory an application is using; Butt answered that this feature was already being worked on.
