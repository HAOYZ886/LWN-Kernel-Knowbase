---
title: "The power-aware scheduling miniconference"
url: https://lwn.net/Articles/609561/
date: "August 27, 2014"
category: "Power management-CPU scheduling; Scheduler-and power management"
author: "By Jonathan Corbet August 27, 2014 Kernel Summit 2014"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 27, 2014

* * *

[Kernel Summit 2014](<https://lwn.net/Articles/KernelSummit2014/>)

For the second year in a row, the annual Kernel Summit included a miniconference on the problem of power-aware scheduling — how can the scheduler place processes so that their execution consumes a minimal amount of power? Morten Rasmussen provided a summary of this year's meeting to the Summit as a whole; it seems clear that a full solution to this problem is still distant, but some progress is slowly being made. 

One of the results from [the 2013 meeting](<https://lwn.net/Articles/571414/>) was that the power-aware scheduling developers needed to come up with a set of metrics and benchmarks so that the results of their work could be judged. Without such metrics, there is no way to know if power-aware scheduling patches are actually achieving their objective. This year, two tools developed by Linaro are being put into use. The first of those is a workload generation tool which can be used to run specific scheduling algorithms and watch the results. A test pattern can be described in JSON and run in the system. There are two workloads described currently: audio playback on an Android system and a generic web-browsing workload. 

The other tool is called "idlestat." It works from data obtained with ftrace on running systems to generate statistics on the sleep states entered by the processor and how long is spent in each. It can accept a power model for a given processor describing the power requirements for each processor state. That model can then be used to generate an estimate of the total amount of power consumed by a given run. 

These tools are a good start, Morten said, but they are just a start. There is a need for community feedback on how well they describe the scheduling problem, and the developers would very much like to have more [![\[Morten Rasmussen\]](https://static.lwn.net/images/conf/2014/ks/MortenRasmussen-sm.jpg)](<https://lwn.net/Articles/609563/>) workload descriptions. For now, though, Morten cautioned, this work is being limited to CPU power consumption. That is a hard enough problem to solve without also trying to address power consumption in the graphics processor or other peripheral devices. 

The [load tracking](<https://lwn.net/Articles/531853/>) that has been added to the scheduler is helpful, Morten said, but it turns out that power-aware scheduling also needs good CPU-utilization tracking, which is a bit different. With utilization tracking, the scheduler can come up with better estimates of how much CPU time each process will require in the future and use those estimates to make better scheduling decisions. Load tracking is also entirely unaware of CPU frequency scaling, a problem that must be fixed. The next step in that direction is to start to control CPU frequency scaling from the scheduler itself, rather than trying to react to what an independent CPU frequency governor is doing. 

Morten also noted that energy awareness will always have to be an optional feature for the scheduler. Some hardware wants to manage power awareness internally with a bunch of "magic behind the scenes." If that functionality cannot be turned off, the hardware is simply not going to be easy to cooperate with in this area; it's better to just let it have its way. 

Developers are currently working on an [energy-model-driven scheduling proposal](<https://lwn.net/Articles/600419/>) that tries to improve the scheduler's decision-making. The task has proved to be more challenging than expected; some relatively simple techniques like [small-task packing](<https://lwn.net/Articles/520857/>) make sense sometimes, but not always. So some way of choosing between scheduling algorithms must be arrived at. One could do it with heuristics, Morten said, but that is painful in the long run. Heuristics are always, at best, an approximation of a complete solution. 

The alternative is to put a model of the CPU platform into the scheduler itself. For any given configuration of processes in the system, the model can generate an estimate of what the energy cost will be. That allows the scheduler to contemplate moving processes around and estimate what the resulting power consumption will be. The platform model must be supplied by architecture-specific code; it will be based on processor idle and sleep states. There is a patch set in circulation, and there appears to be a consensus around this approach, with no major objections being expressed. 

A future task, Morten said, will be to move CPU idle-state awareness into the scheduler. Like frequency scaling, the CPU idle-management code runs as a separate subsystem with no direct communication with the scheduler. Bringing idle awareness into the power model will allow the scheduler to better manage idle time and to make better predictions of future wakeup events. 

Another future-work area is the management of power policies under virtualization. Guest systems, too, want to run in a power-efficient manner. The consensus seems to be, though, that power management should be handled entirely in the host. Guests can communicate their constraints to the hypervisor, but any attempt to implement those constraints belongs on the host side. 

As Morten's report came to a close, a developer asked whether the power-aware scheduling developers were working on thermal awareness as well. That topic came up during the miniconference, Morten said, but it is not being worked on at the moment. The power model is being kept as simple as possible for now; the developers feel like they have enough complexity to deal with as it is. Once it appears that a solution to the simpler problem is in sight, they can consider taking on additional constraints like thermal management.
