---
title: "Tracking pressure-stall information"
url: https://lwn.net/Articles/759781/
date: "July 13, 2018"
category: Performance monitoring
author: "By Jonathan Corbet July 13, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 13, 2018

All underutilized systems are essentially the same, but each overutilized system tends to be overloaded in its own way. If one's goal is to maximize the use of the available computing resources, overutilization tends not to be too far away, but when it happens, it can be hard to tell where the problem is. Sometimes, even the fact that there is a problem at all is not immediately apparent. [The pressure-stall information patch set](<https://lwn.net/ml/cgroups/20180712172942.10094-1-hannes@cmpxchg.org/>) from Johannes Weiner may make life easier for system administrators by exposing more information about the real utilization state of the system. 

A kernel with this patch set applied will have a new virtual directory called `/proc/pressure` containing three files. The first, `cpu`, describes the state of CPU utilization in the system. Reading it will produce a line like this: 

```
some avg10=2.04 avg60=0.75 avg300=0.40 total=157656722
```

The `avg` numbers give the percentage of the time that runnable processes are delayed because the CPU is unavailable to them, accumulated over ten, 60, and 300 seconds. In a system with just one runnable process per CPU, the numbers will all be zero. If those numbers start to increase significantly, that means that processes are running more slowly than they otherwise would due to overloading of the CPUs. Administrators can use this information to determine whether the amount of delay due to CPU contention is within the bounds they can tolerate or whether something must be done to ensure that things run more quickly. 

These delay numbers resemble the system load average, in that they both give a sense for how busy the system is. The load average is simply the _number_ of processes waiting for the CPU (along with those in short-term I/O waits), though; it needs to be interpreted relative to the number of available CPUs to have meaning. The stall information, instead, tracks the actual amount of waiting time. It is also tracked over a much shorter time range than the load average. 

The final number (`total`) is the total amount of time (in microseconds) during which processes were stalled. It is there to help with the detection of short-term latency spikes that wouldn't show up in the aggregated numbers. A system where a CPU is nearly always available but where occasional 10ms latency spikes are experienced may be entirely acceptable for some workloads, but not for others. For the latter group, the `total` count can be monitored to detect when those spikes are happening. 

The next file is `/proc/pressure/memory`; as might be expected, it provides information on the time that processes spend waiting due to memory pressure. Its output looks like: 

```
some avg10=70.24 avg60=68.52 avg300=69.91 total=3559632828
        full avg10=57.59 avg60=58.06 avg300=60.38 total=3300487258
```

The `some` line is similar to the CPU information: it tracks the percentage of the time that at least one process could be running if it weren't waiting for memory resources. In particular, the time spent for swapping in, refaulting pages from the page cache, and performing direct reclaim is tracked in this way. It is, thus, a good indicator of when the system is thrashing due to a lack of memory. 

The `full` line is a little different: it tracks the time that nobody is able to use the CPU for actual work due to memory pressure. If all processes are waiting for paging I/O, the CPU may look idle, but that's not because of a lack of work to do. If those processes are performing memory reclaim, the end result is nearly the same; the CPU is busy, but it's not doing the work that the computer is there to do. If the `full` numbers are much above zero, it's clear that the system lacks the memory it needs to support the current workload. 

Some care has been taken to distinguish paging due to thrashing from other sorts of paging. A process that is just starting up will experience a lot of page faults as its working set is brought in, but those are not really indicative of system load. For that reason, _refaulted_ pages — those which were evicted due to memory pressure and subsequently brought back in — are used to calculate these metrics (see [this article](<https://lwn.net/Articles/495543/>) for a description of how refaults are tracked). Even then, though, there is a twist, in that a process may need different sets of pages during different phases of its execution. To try to detect the transition between different working sets, the patch set adds tracking of whether each page has made it to the active list (was used more than once, essentially) since it was faulted in. Only the pages that are actually used are counted when the stall times are calculated. 

The final file is `/proc/pressure/io`, which tracks the time lost waiting for I/O. This number is likely to be more difficult to make good use of without some sense for what the baseline values should be. The block subsystem isn't able to track the amount of extra time spent waiting due to contention for the device, so the resulting numbers will not be directly related to that contention. 

The files in `/proc/pressure` track the state of the system as a whole. In systems where control groups are in use, there will also be a set of files (`cpu.pressure`, `memory.pressure`, and `io.pressure`) associated with each group. They can be used to ensure that the resource limits for each group make sense; they should also make it easier to determine which processes are thrashing on a busy system. 

This functionality has apparently been used within Facebook for some time, and has helped considerably in the optimization of system resources and the diagnosis of problems. ""We now log and graph pressure for the containers in our fleet and can trivially link latency spikes and throughput drops to shortages of specific resources after the fact, and fix the job config/scheduling"", Weiner said. There is also evidently interest from the Android world, where developers are looking for better ways of detecting out-of-memory situations before system performance is entirely lost. Linus Torvalds has [indicated](<https://lwn.net/ml/cgroups/CAADWXX_ADRyY+HDyw_2Ofa4b9g1fpCNe8xO3xMf20jfWsyCnQg@mail.gmail.com/>) that the idea looks interesting to him. There are still some open questions on how the CPU data is accumulated (see [this message](<https://lwn.net/ml/cgroups/20180713161756.GA21168@cmpxchg.org/>) for a long explanation), but one assumes that will be worked out before too long. So, in all likelihood, the pressure-stall patches will not be stalled for too long before making it into the mainline.
