---
title: "Another attempt at power-aware scheduling"
url: https://lwn.net/Articles/600419/
date: "June 4, 2014"
category: "Power management-CPU scheduling; Scheduler-and power management"
author: "By Jonathan Corbet June 4, 2014"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 4, 2014

Numerous attempts to improve the power efficiency of the kernel's CPU scheduler have been made in recent years. Most of these attempts have taken the form of adding heuristics to the scheduler ("group small tasks onto just a few CPUs," for example) that, it was hoped, would lead to more efficient use of the system's resources. These attempts have run aground for a number of reasons, including the fact that they tend to work for only a subset of the range of workloads and systems out there and their lack of integration with other CPU-related power management subsystems, including the CPU frequency and CPU idle governors. At the [power-aware scheduling mini-summit](<https://lwn.net/Articles/571414/>) in 2013, a call was made for a more organized approach to the problem. Half a year or so later, some of the results are starting to appear. 

In particular, Morten Rasmussen's [Energy cost model for energy-aware scheduling patch set](<https://lwn.net/Articles/600135/>) was posted on May 23\. This patch set is more of a demonstration-of-concept than something suitable for merging, but it does show the kind of thinking that is going into power-aware scheduling now. Heuristics have been replaced with an attempt to measure and calculate what the power cost of each scheduling decision will be. 

The patch set starts by creating a new data structure to describe the available computing capacity of each CPU and the power cost of running at each capacity. If a given CPU can operate at three independent frequencies, this data structure will contain a three-element array describing the power cost of running at each frequency and the associated computing capacity that will be available. There are no specific units associated with either number; as long as they are consistent across the system, things will work. 

On a simple system, the cost/capacity array will be the same for each CPU. But things can quickly get more complicated than that. Asymmetric systems (big.LITTLE systems, for example) have both low-power and high-power CPUs offering a wide range of capacities. On larger systems, CPUs are organized into packages and NUMA nodes; the power cost of running two CPUs on the same package will be quite a bit less than the cost of keeping two packages powered up. So the cost/capacity array must be maintained at every level of the [scheduling domain](<https://lwn.net/Articles/80911/>) hierarchy (which matches the hardware topography), and scheduling decisions must take into account the associated cost at every level. 

In the current patch set, this data structure is hard coded for a specific ARM processor. One of the many items on the "to do" list is to create this data structure automatically, either from data found in the firmware or from a device tree. Either way, some architecture-specific code will have to be written, but that was not a problem that needed to be solved to test out the concepts behind this patch set. 

With this data structure in place, it is possible to create a simple function: 

```
int energy_diff_util(int cpu, int utilization);
```

The idea is straightforward enough: return the difference in power consumption that will result from adding a specific load (represented by `utilization`) to a given CPU. In the real world, though, there are a few difficulties to be dealt with. One of those is that the kernel does not really know how much CPU utilization comes with a specific task. So the patch set has to work with the measured load values, which are not quite the same thing; in particular, load does not take a process's priority into account. 

Then there is the little problem that the scheduler does not actually know anything about what the CPU frequency governor is doing with any given CPU. The patch set adds a hack to make the current frequency of each CPU available, and there is an explicit assumption that the governor will make changes to match utilization changes on any given processor. The lack of integration between these subsystems was a major complaint at last year's mini-summit; it is clearly a problem that will need to be addressed as part of any viable power-aware scheduling patch. But, for the time being, it's another detail that can be glossed over while the main concepts are worked out. 

There are a number of factors beyond pure CPU usage that can change how much power a given process needs. One of those is CPU wakeups: taking a processor out of a sleep state has an energy cost of its own. It is not possible to know how often a given process will awaken a sleeping CPU, but one can get an approximate measure by tracking how often the process itself wakes up from a sleeping state. If one assumes that some percentage of those wakeups will happen when the CPU itself was sleeping, one can make a guess at how many CPU wakeups will be added if a process is made to run on a given CPU. 

So Morten's patch set adds simple process wakeup tracking to get a sense for whether a given process wakes up frequently or rarely. Then, when the time comes to consider running that process on a given CPU, a look at that CPU's current idle time will generate a guess for how many additional wakeups the process would create there. A CPU that is already busy most of the time will not sleep often, so it will suffer fewer wakeups than one that is mostly idle. Factor in the energy cost of waking the CPU (which will depend on just how deeply it is sleeping, another quantity that is hard for the scheduler to get currently) and an approximate energy cost associated with wakeups can be calculated. 

With that structure in place, it's just a matter of performing the energy calculations for each possible destination when the time comes to pick a CPU for a given task. Iterating through all CPUs could get expensive, so the code tries to quickly narrow things down to one low-level group of CPUs; the lowest-cost CPU in that group is then chosen. In this patch set, `find_idlest_cpu()` is modified to do this search; other places where task placement decisions are made (load balancing, for example) have not been modified. 

The patch set came with a small amount of benchmark information; it shows energy savings from 3% to 50%, depending on the workload, on a big.LITTLE system. As Morten notes, the savings on a fully symmetric system will be smaller. There is also an approximate quadrupling of the time taken to switch tasks; that cost is likely to be seen as unacceptable, but it should also be possible to reduce that cost considerably with some targeted optimization work. 

Thus far, discussion of the patch set has been muted. Getting sufficient reviewer attention on power-aware scheduling patches has been a problem in the past. The tighter focus of this patch set should help to make review relatively easy, though, so, with luck, this work will be looked over in the near future. Then we'll have an idea of whether it represents a viable path forward or not.
