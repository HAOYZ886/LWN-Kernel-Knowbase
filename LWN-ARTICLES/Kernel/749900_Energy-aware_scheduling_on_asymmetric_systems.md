---
title: "Energy-aware scheduling on asymmetric systems"
url: https://lwn.net/Articles/749900/
date: "March 22, 2018"
category: "Power management-CPU scheduling; Scheduler-and power management"
author: "By Jonathan Corbet March 22, 2018"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 22, 2018

Energy-aware scheduling — running a system's workload in a way that minimizes the amount of energy consumed — has been a topic of active discussion and development for some time; LWN first [covered the issue](<https://lwn.net/Articles/474915/>) at the beginning of 2012. Many approaches have been tried during the intervening years, but little in the way of generalized energy-aware scheduling work has made it into the mainline. Recently, [a new patch set](<https://lwn.net/Articles/749738/>) was posted by Dietmar Eggemann that only tries to address one aspect of the problem; perhaps the problem domain has now been simplified enough that this support can finally be merged. 

In the end, the scheduler can most effectively reduce power consumption by keeping the system's CPUs in the lowest possible power states for the longest time — with "sleeping" being the state preferred over all of the others. There is a tradeoff, though, in that users tend to lack appreciation for saved power if their systems are not responsive; any energy-aware scheduling solution must also be aware of throughput and latency concerns. A failure to balance all of these objectives across the wide range of machines that run Linux has been the bane of many patches over the years. 

There have been a number of clever ideas that have been attempted, of course. [Small-task packing](<https://lwn.net/Articles/520857/>) tries to group small, sporadic processes onto a small number of CPUs to prevent them from waking the others. [Other patch sets](<https://lwn.net/Articles/546664/>) have used a spreading technique in an attempt to evacuate CPUs with relatively low loads. There has been talk of a separate [power scheduler](<https://lwn.net/Articles/570353/>) whose job is to run each CPU at the optimal power level for the current workload. The [energy cost model](<https://lwn.net/Articles/600419/>) created a data structure to track the performance and energy cost of each processor state and used it to inform scheduling decisions. The [SchedTune](<https://lwn.net/Articles/706374/>) CPU-frequency governor allows some tasks to be designated as "important", with the less-important ones being relegated to low amounts of CPU power. Some of these ideas have influenced the mainline scheduler but, as a whole, they remain outside. 

Saving energy is valuable in almost every setting from tiny embedded systems to supercomputer installations. But the pressure tends to be most acutely felt in the area of mobile systems; the less power a device uses, the longer it can run before exhausting its battery. It is thus not surprising that most of the energy-aware scheduling work has been driven by the mobile market. The Android Open Source Project's kernel includes a version of the energy-aware scheduler patches; those have been shipping on handsets for some time. Scheduling, as a result, is one of the areas where the Android and mainline kernels differ the most. 

Eggemann's patch set is intended to reduce that difference by proposing a simplified version of the Android scheduler. To that end, it only addresses the problem for asymmetric systems — those with CPUs that have varying power characteristics, such as the ARM big.LITTLE processors. Since the "little" processors are much more energy-efficient (but much slower) than the "big" ones, the placement of processes in the system can have a significant effect on both energy consumption and performance. Improving task placement under mainline kernels on big.LITTLE systems is arguably the most urgent problem in the energy-aware scheduling area. 

To get there, the patch set adds a simplified version of the energy-cost model used in the Android scheduler. It is defined entirely with these two structures: 

```
struct capacity_state {
    	unsigned long cap;	/* compute capacity */
    	unsigned long power;	/* power consumption at this compute capacity */
        };
    
        struct sched_energy_model {
    	int nr_cap_states;
    	struct capacity_state *cap_states;
        }
```

The units of both `cap` and `power` are not really defined, but they do not need to be as long as they are used consistently across the CPUs of the system. There is one `capacity_state` structure for each power state of each CPU, so the scheduler can immediately determine what the cost (or benefit) of changing a given CPU's state would be. Each CPU has a `sched_energy_model` structure containing the data for all of its available power states. 

This information, as it turns out, is already available in some systems at least, since the thermal subsystem makes use of it to help keep the system from overheating. That is a useful attribute; it means that a scheduler with these patches could be run on existing hardware without the need to provide more information (through device-tree entries, for example). 

The scheduler already performs load tracking, which allows it to estimate how much load each process will put on a CPU when it is run there. That load estimate is used along with the energy model to determine where a task should run when it wakes up. This is done by looking at each CPU in the [scheduling domain](<https://lwn.net/Articles/80911/>) where the process last ran and determining what the energy cost of placing the process on each CPU would be. Essentially, if the CPU would have to go to a higher power state to run the added load in a timely manner, the cost would be the additional energy needed to sustain that higher state. In the end, the CPU with the lowest added cost is the one the will run the new process. 

The process wakeup path is rather performance-critical, so the above algorithm raises some red flags. Iterating over every CPU in the system (or even just a subset in a given domain) could become quite expensive in a system with a lot of CPUs. This algorithm is only enabled on asymmetric systems, which minimizes that cost because such systems (currently) have a maximum of eight CPUs. Those also are the systems that benefit most from this sort of energy-use calculation. Data-center systems with large numbers of identical CPUs would see little improvement from this approach, so it is not enabled there. 

Even on asymmetric systems, though, this algorithm will not help if the system is already running near its capacity; in that case, the CPUs will already be running at a high power point and there is little value to be had from looking at power costs. If the scheduling domain where the process last ran is determined to be "overutilized", defined as running at 80% of its maximum capacity or higher, then the current wakeup path (which tries to find the most lightly loaded CPU) is used instead. 

Some benchmarks posted with the patch set show some significant energy-use improvements with the patches applied — up to 33% in one case. There is a small cost in throughput (up to about 2% in one test, but usually much lower) that comes with that improvement. That is a cost that most mobile users are likely to be willing to pay for that kind of battery-life improvement. 

Discussion of the patch set has mostly been focused on implementation details so far, and there has not yet been input from the core scheduler maintainers. So there is no way to really know whether this approach has a better chance of getting over the acceptance hurdle than its predecessors. Given that it is relatively simple and the costs are only paid on systems that benefit from this algorithm, though, one might expect that its chances would be relatively good. Acceptance would not unify the mainline and Android schedulers, but it would be a big step in the right direction.
