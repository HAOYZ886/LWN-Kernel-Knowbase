---
title: Sched_ext at LPC 2024
url: https://lwn.net/Articles/991205/
date: "September 26, 2024"
category: "BPF-CPU scheduling; Scheduler-Extensible scheduler class"
author: "By Jonathan Corbet September 26, 2024 LPC"
---

By **Jonathan Corbet**  
September 26, 2024

* * *

[LPC](<https://lwn.net/Archives/ConferenceIndex/#Linux_Plumbers_Conference-2024>)

The [extensible scheduler class (sched_ext)](<https://lwn.net/Articles/922405/>) enables the implementation of CPU schedulers as a set of BPF programs loaded from user space; it first hit the mailing lists in late 2022. Sched_ext has engendered its share of controversy since, but is currently slated to be part of the 6.12 kernel release. At the 2024 [Linux Plumbers Conference](<https://lpc.events/>), the growing sched_ext community held one of its first public gatherings; sched_ext would appear to have launched a new burst of creativity in scheduler design. 

#### An overview

[![\[Tejun Heo\]](https://static.lwn.net/images/conf/2024/lpc/TejunHeo-sm.png)](<https://lwn.net/Articles/991293/>) The [sched_ext microconference](<https://lpc.events/event/18/sessions/192/#20240918>) began with Tejun Heo, one of the authors of this new subsystem. He introduced sched_ext as a new scheduling class that sits in the hierarchy along with the [EEVDF](<https://lwn.net/Articles/969062/>) and realtime schedulers. It serves as a bridge between the scheduling core and the BPF virtual machine, where all of the interesting decisions are made. BPF maps are used as a high-performance interface to user space. 

Work on sched_ext is proceeding on several fronts, starting with the merge of the sched_ext core into the mainline kernel, which was still pending at the time of this talk. Basic scheduling is part of that and works now. There are ongoing efforts to support features like CPU-frequency scaling, CPU hotplug, and control groups (basic support for which landed in 6.12); those can be expected to be added in future mainline kernel releases. 

There is [a repository](<https://github.com/sched-ext/scx>) available with the current sched_ext work, including a number of example schedulers. The `scx_lavd` scheduler, for example, is focused on interactivity and, specifically, consistently getting higher frame rates out of games. The `scx_bpfland` scheduler, instead, is aimed at minimizing response times. There is `scx_rustland`, which simply forwards scheduling events to user space, where the decisions are made. Heo admitted that he had been skeptical of that idea at the outset, but it has turned out to be "quite usable". The repository also contains `scx_rusty` for load balancing on complex CPU topologies, and `scx_layered`, which is a partitioning scheduler. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

One of the best things about sched_ext, he said, is that it cannot crash the machine. All of the usual BPF safety checks apply here. Additionally, if the kernel detects a scheduling problem, it will simply revert the system to the EEVDF scheduler and life goes on. That makes experimenting easy and the development cycle short. 

It is, he said, still the early days of sched_ext development, and he is focused on getting some practical wins. One of those appears to be `scx_lavd` (about which more was heard later), which is headed for shipment in [Steam Deck](<https://store.steampowered.com/steamdeck>) gaming systems. `scx_bpfland` is showing promising results for personal machines, while `scx_layered` has been deployed in over one million machines and is delivering significant performance gains. 

The sched_ext developers are also working on building the development community. Support for sched_ext is now shipping in a number of distributions, including CachyOS, Arch Linux, Ubuntu, Fedora, Nix, and openSUSE. On those distributions, running a new scheduler is just a matter of installing a package and running a program. 

Work in the future, Heo said in conclusion, is focused on composability — making it possible for multiple schedulers to work together. That will allow different developers to focus on independent layers; one scheduler could be concerned with time slices, while another would focus on load balancing or partitioning. The plan is also to eventually allow the stacking of schedulers down the control-group hierarchy, so that different schedulers at each level could handle a part of the overall scheduling problem. 

#### User-space scheduling

[![\[Andrea Righi\]](https://static.lwn.net/images/conf/2024/lpc/AndreaRighi-sm.png)](<https://lwn.net/Articles/991294/>) While sched_ext is meant to put CPU-scheduling decisions into users' hands, it was still expected that those decisions would be made by BPF programs running within the kernel. So Andrea Righi's `scx_rustland` scheduler, which defers all of those decisions to user space, came as a bit of a surprise. Righi started his session by saying that `scx_rustland` began as just a fun project, with no expectation that something useful would result. He mostly wanted better observability of scheduling decisions and a faster development cycle, where installing a new scheduler is just a matter of restarting a program. 

What he came up with is a new Rust crate providing the scheduling interface; it is licensed under GPLv2. Schedulers are thus written in Rust, but the BPF code, which mostly just communicates events and decisions between the kernel and user space, is still compiled from C code. A pair of ring buffers is used for communication; initially BPF maps had been used, but the ring buffers are much faster. The API for schedulers has been deliberately kept simple, with the idea that anybody should be able to use it to write a new scheduler. 

Righi admitted, though, that `scx_rustland` "is not all rainbows and unicorns". One significant problem is that the scheduler program cannot block for any reason (such as a page fault), or scheduling as a whole comes to a halt. So a custom memory allocator is used to keep the scheduler running in locked-in-RAM memory. Multithreading in the scheduling program is "tricky" but mostly solved. Even with the ring buffers, the communication overhead with the kernel is significant, but not a huge issue. There are some possible sched_ext changes that would help there. 

Righi's future plans include standardizing and locking down the user-space API for schedulers. He would also like to create a concept of "scheduling domains", each of which is made up of a set of CPUs. The ability to attach a task to one of these domains would make scheduling easier and improve performance. 

#### Higher frame rates

Changwoo Min took over via a remote link to talk about `scx_lavd`, which is a "latency criticality aware virtual deadline" scheduler aimed at gaming applications. It uses latency criticality (described later) as the primary scheduling decision, handles heterogeneous cores well, and adapts its scheduling decisions to the load pattern on the system. 

The goal behind this scheduler was to provide the best gaming experience on Linux in general — not just on the Steam Deck. That requires getting high performance (and high video frame rates) without stuttering (short-term performance loss due to load in the system). The scheduler should deliver reasonable performance across a wide range of CPU configurations, but it is not intended to be the best server or general-purpose scheduler. 

A key aspect of gaming workloads is that tasks tend to run quickly, typically no more than 100µs at a time. There are a lot of tightly linked tasks, though, and performance depends on the most critical of those tasks running in the necessary sequence; that is the critical path. Every task has a latency criticality that is determined by its place in this path; tasks that wait on others, and are waited on in turn, have a large impact on overall performance and are thus "latency critical". Detecting these tasks requires observing which tasks wait for which others, and ensuring that the tasks being waited for are run with low latency. 

Each task has a virtual deadline calculated for it, which is a function of both its waking and waiting frequencies — its latency criticality, in other words. Tasks that both wait often for others and are often waited upon are seen as the most critical, so their deadline is the shortest. Time slices are then assigned in a manner similar to how the completely fair scheduler does it; slices are fixed, but get shorter as the number of runnable tasks increases. 

Care is also taken to chose CPUs properly on heterogeneous systems. At times of low load, with a simple workload, the low-power cores can get the job done while minimizing power use. If the load is heavy, though, then performance becomes the primary goal, and the fast cores must be used. The in-between case is trickier; some tasks can be put on smaller cores, but some will need the faster ones. 

In the `scx_lavd` "autopilot" mode, the scheduler looks at the current CPU utilization. For light loads, a power-saving mode is chosen; for heavy loads, the fast cores are used in a race-to-idle strategy. In between those extremes, the scheduler tries to minimize the number of cores in use, but takes care to put the latency-critical tasks onto the large cores. 

Min concluded by saying that, for gaming applications, `scx_lavd` consistently enables higher frame rates than the EEVDF scheduler while using (slightly) less power and with fewer stutters. 

#### A lot of activity

The sched_ext microconference included a number of other presentations, some from people who had been working on out-of-tree schedulers for years. Barret Rhoden and Josh Don talked about the use of pluggable scheduling within Google, a project that has been underway since 2019. Once again, this effort was able to obtain better performance, but also highlighted the fact that different workloads benefit from different scheduling policies. Himadri Chhaya-Shailesh discussed using sched_ext for paravirtualized scheduling, where host and guest schedulers communicate to optimize the overall result. Masahito Suzuki and Alfred Chen have both been working on out-of-tree schedulers for desktop use. Peter Jung discussed the [CachyOS](<https://cachyos.org/>) distribution, which has been shipping a range of out-of-tree schedulers for years; developers there have created a whole infrastructure allowing users to switch schedulers on the fly. 

The kernel project has long had a policy that it would support one general-purpose CPU scheduler, and that scheduler had to provide good service for all workloads. This policy has, beyond a doubt, resulted in a sophisticated scheduler that is able to run on everything from small embedded systems to massive data-center machines. It has ensured that all users benefit from scheduler improvements. 

What was made abundantly clear at the sched_ext microconference, though, is that this policy has also led to the marginalization of a lot of creative work in this area. A scheduler that cannot regress for any workload leaves little room for developers wanting to optimize a specific class of applications, and who cannot even test many other workloads. This is a hard area in which to scratch an itch; developers have been discouraged from trying, and those who have ventured into this area have rarely seen their work enter the mainline kernel. 

Sched_ext has removed many of the barriers to entry in the area of scheduler development, and the result has been an immediate increase in the number of developers playing with ideas and seeing where they lead. There is a new community that is quickly forming here, and it seems likely to come up with some novel (and sometimes crazy) approaches to CPU scheduling. This will be an interesting space to watch in the coming years. 

[ Thanks to the Linux Foundation, LWN's travel sponsor, for supporting our travel to this event. ]
