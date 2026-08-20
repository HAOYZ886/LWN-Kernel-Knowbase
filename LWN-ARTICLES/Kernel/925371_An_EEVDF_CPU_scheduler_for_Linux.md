---
title: An EEVDF CPU scheduler for Linux
url: https://lwn.net/Articles/925371/
date: "March 9, 2023"
category: "Releases-6.6; Scheduler-Completely fair scheduler; Scheduler-EEVDF"
author: "By Jonathan Corbet March 9, 2023"
---

By **Jonathan Corbet**  
March 9, 2023

The kernel's [completely fair scheduler (CFS)](<https://lwn.net/Articles/230574/>) has the job of managing the allocation of CPU time for most of the processes running on most Linux systems. CFS was merged for the 2.6.23 release in 2007 and has, with numerous ongoing tweaks, handled the job reasonably well ever since. CFS is not perfect, though, and there are some situations it does not handle as well as it should. The [EEVDF scheduler](<https://lwn.net/ml/linux-kernel/20230306132521.968182689@infradead.org/>), posted by Peter Zijlstra, offers the possibility of improving on CFS while reducing its dependence on often-fragile heuristics. 

#### CFS and scheduling constraints

One of the key design goals of CFS was, as might be understood from its name, fairness — ensuring that every process in the system gets its fair share of CPU time. This goal is achieved by tracking how much time each process has received and running those that have gotten less CPU time than the others, with each process's run time scaled by its "nice" priority. CFS is, in other words, a [weighted fair queuing](<https://en.wikipedia.org/wiki/Weighted_fair_queueing>) scheduler at its core. 

Fairness, it turns out, is enough to solve many CPU-scheduling problems. There are, however, many constraints beyond the fair allocation of CPU time that are placed on the scheduler. It should, for example, maximize the benefit of the system's memory caches, which requires minimizing the movement of processes between CPUs. At the same time, though, it should keep all CPUs busy if there is work for them to do. Power management is a complication as well; sometimes the optimal decisions for system throughput must take a back seat to preserving battery life. Hybrid systems (where not all CPUs are the same) add [more complications](<https://lwn.net/Articles/909611/>). And so on. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

One place where there is a desire for improvement is in the handling of latency requirements. Some processes may not need a lot of CPU time but, when they _do_ need that time, they need it quickly. Others might need more CPU time but can wait for it if need be. CFS does not give processes a way to express their latency requirements; nice values (priorities) can be used to give a process more CPU time, but that is not the same thing. The realtime scheduling classes _can_ be used for latency-sensitive work, but running in a realtime class is a privileged operation, and realtime processes can badly affect the operation of the rest of the system. 

What is lacking is a way to ensure that some processes can get access to a CPU quickly without necessarily giving those processes the ability to obtain more than their fair share of CPU time. The [latency nice](<https://lwn.net/Articles/887842/>) patches have been circulating for some time as an attempt to solve this problem; they allow CFS processes with tight latency requirements to jump the queue for the CPU when they want to run. These patches appear to work, but Zijlstra thinks that there might be a better approach to the problem. 

#### Introducing EEVDF

The "Earliest Eligible Virtual Deadline First" (EEVDF) scheduling algorithm is not new; it was described in [this 1995 paper](<https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=805acf7726282721504c8f00575d91ebfd750564>) by Ion Stoica and Hussein Abdel-Wahab. Its name suggests something similar to the Earliest Deadline First algorithm used by the kernel's [deadline scheduler](<https://lwn.net/Articles/743740/>) but, unlike that scheduler, EEVDF is not a realtime scheduler, so it works in different ways. Understanding EEVDF requires getting a handle on a few (relatively) simple concepts. 

Like CFS, EEVDF tries to divide the available CPU time fairly among the processes that are contending for it. If, for example, there are five processes trying to run on a single CPU, each of those processes should get 20% of the available time. A given process's nice value can be used to adjust the calculation of what its fair time is; a process with a lower nice value (and thus a higher priority) is entitled to more CPU time at the expense of those with higher nice values. To this point, there is nothing new here. 

Imagine a time period of one second; during that time, in our five-process scenario, each process should have gotten 200ms of CPU time. For a number of reasons, things never turn out exactly that way; some processes will have gotten too much time, while others will have been shortchanged. For each process, EEVDF calculates the difference between the time that process should have gotten and how much it actually got; that difference is called "lag". A process with a positive lag value has not received its fair share and should be scheduled sooner than one with a negative lag value. 

In fact, a process is deemed to be "eligible" if — and only if — its calculated lag is greater than or equal to zero; any process with a negative lag will not be eligible to run. For any ineligible process, there will be a time in the future where the time it is entitled to catches up to the time it has actually gotten and it will become eligible again; that time is deemed the "eligible time". 

The calculation of lag is, thus, a key part of the EEVDF scheduler, and much of the patch set is dedicated to finding this value correctly. Even in the absence of the full EEVDF algorithm, a process's lag can be used to place it fairly in the run queue; processes with higher lag should be run first in an attempt to even out lag values across the system. 

The other factor that comes into play is the "virtual deadline", which is the earliest time by which a process should have received its due CPU time. This deadline is calculated by adding a process's allocated time slice to its eligible time. A process with a 10ms time slice, and whose eligible time is 20ms in the future, will have a virtual deadline that is 30ms in the future. 

The core of EEVDF, as can be seen in its name, is that it will run the process with the earliest virtual deadline first. The scheduling choice is thus driven by a combination of fairness (the lag value that is used to calculate the eligible time) and the amount of time that each process currently has due to it. 

#### Addressing the latency problem

With this framework in place, the implementation of quicker access for latency-sensitive processes happens naturally. When the scheduler is calculating the time slice for each process, it factors in that process's assigned latency-nice value; a process with a lower latency-nice setting (and, thus, tighter latency requirements) will get a shorter time slice. Processes that are relatively indifferent to latency will receive longer slices. Note that the _amount_ of CPU time given to any two processes (with the same nice value) will be the same, but the low-latency process will get it in a larger number of shorter slices. 

Remember that the virtual deadline is calculated by adding the time slice to the eligible time. That will cause processes with shorter time slices to have closer virtual deadlines and, as a result, to be executed first. Latency-sensitive processes, which normally don't need large amounts of CPU time, will be able to respond quickly to events, while processes without latency requirements will be given longer time slices, which can help to improve throughput. No tricky scheduler heuristics are needed to get this result. 

There is a big distance, though, between an academic paper and an implementation that can perform well in the Linux kernel. Zijlstra has only begun to run benchmarks on his EEVDF scheduler; his initial conclusion is that ""there's a bunch of wins and losses, but nothing that indicates a total fail"". Some of the results, he said, ""seem to indicate EEVDF schedules a lot more consistently than CFS and has a bunch of latency wins"". 

While this is clearly a reasonable starting point, Zijlstra acknowledges that there is still quite a bit of work to be done. But, he said, ""if we can pull this off we can delete a whole [bunch] of icky heuristics code"", replacing it with a better-defined policy. This [is not a small change](<https://lwn.net/ml/linux-kernel/20230307130800.GD2017917@hirez.programming.kicks-ass.net/>), he added: ""It completely reworks the base scheduler, placement, preemption, picking -- everything. The only thing they have in common is that they're both a virtual time based scheduler."" 

Needless to say, such a fundamental change is unlikely to be rushed into the kernel. Helpfully, the current patches implement EEVDF as an option alongside CFS, which will enable wider testing without actually replacing the current scheduler. The CPU scheduler has to do the right thing for almost any conceivable workload on the wide range of systems supported by the kernel; that leaves a lot of room for unwelcome regressions resulting from even small changes — which this is not. So a lot of that testing will have to happen before consideration might be given to replacing CFS with EEVDF; there is no deadline, virtual or otherwise, for when that might happen.
