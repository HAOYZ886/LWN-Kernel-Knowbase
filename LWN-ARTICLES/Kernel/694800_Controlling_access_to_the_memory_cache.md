---
title: Controlling access to the memory cache
url: https://lwn.net/Articles/694800/
date: "July 20, 2016"
category: Resource management
author: "By Jonathan Corbet July 20, 2016 LinuxCon Japan"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 20, 2016

* * *

[LinuxCon Japan](<https://lwn.net/Archives/ConferenceByYear/#2016-LinuxCon_Japan>)

Access to main memory from the processor is mediated (and accelerated) by the L2 and L3 memory caches; developers working on performance-critical code quickly learn that cache utilization can have a huge effect on how quickly an application (or a kernel) runs. But, as Fenghua Yu noted in his LinuxCon Japan 2016 talk, the caches are a shared resource, so even a cache-optimal application can be slowed by an unrelated task, possibly running on a different CPU. Intel has been working on a mechanism that allows a system administrator to set cache-sharing policies; the talk described the need for this mechanism and how access to it is implemented in the current patch set. 

#### Control over cache usage

Yu started off by saying that a shared cache is subject to the "noisy neighbor" problem; a program that uses a lot of cache entries can cause the eviction of entries used by others, hurting their performance. The L3 cache is shared by all CPUs on the same socket, so the annoying neighbor need not be running on the same processor; a cache-noisy program can create problems for others running on any CPU in the socket. A low-priority process that causes cache churn can slow down a much higher-priority process; increased interrupt-response latency is another problem that often results. 

The solution to the problem is to eliminate cache sharing between parts of the system that should be isolated from each other; this is done by partitioning the available cache. Each partition is shared between fewer [![\[Fenghua Yu\]](https://static.lwn.net/images/conf/2016/lcj/FenghuaYu-sm.jpg)](<https://lwn.net/Articles/694872/>) processes and, thus, has fewer conflicts. There is an associated cost, clearly, in that a process running on a partitioned cache has a smaller cache. That, Yu said, can affect the overall throughput of the system, but that is a separate concern. 

Intel's cache-partitioning mechanism is called "cache allocation technology," or CAT. Haswell-generation (and later) server chips have support for CAT at the L3 (socket) level. The documentation also describes L2 (core-level) support, but that feature is not available in any existing hardware. 

In a CAT-enabled processor, it is possible to set up one or more cache bitmaps ("CBMs") describing the portion of the cache that may be used. If, on a particular CPU, the L3 cache is divided into 20 slices, then a CBM of `0xfffff` describes the entire cache, while `0xf8000` and `0x7c00` describe two disjoint regions, each covering 25% of the cache. 

The CBMs are kept in a small table, indexed by a "class of service ID" or CLOSID. The CLOSID will eventually control multiple resources (L2 cache, for example, or something entirely different) but, in current processors, it only selects the active CBM for the L3 cache. At any given time, a specific CLOSID will be active in each CPU, controlling which portion of the cache that CPU can make use of. Each CPU has its own set of CLOSIDs; they are not a system-wide resource. 

Kernel support is needed to make proper use of the CAT functionality. The number of CLOSIDs available is relatively small, so the kernel must arbitrate access to them. Like any resource-allocation technology, CAT control must be limited to privileged users or it will be circumvented. Yu described how CAT policies can be controlled via the interfaces implemented in the current patch set but, before getting into that, it's worthwhile to step away from the talk for a moment and look at the history of this interface. 

#### Unsuccessful kittens

New hardware features often present interesting problems when the time comes to add support to the kernel. It is relatively easy to add that support as a simple wrapper and access-control layer around the feature, but care must be taken to avoid tying the interface to the hardware itself. A vendor's idea of how the feature should work can change over time, and other manufacturers may have ideas of their own. Any interface that is unable to evolve with the hardware will become unsupportable over time and have to be replaced. So it is important to provide an interface that abstracts away the details of how the hardware works to the greatest extent possible. At the same time, though, the interface cannot be so abstract that it makes some important functionality unavailable. 

The first public attempt at CAT support in the kernel appears to be [this patch set](<https://lwn.net/Articles/622893/>) posted by Vikas Shivappa in late 2014. The approach taken was to use the control-group mechanism to set the CBM for groups of processes; the CLOSID mechanism was hidden by the kernel and not visible to user space at all. The initial review discussion focused on some of the more glaring deficiencies in the patch set, so it took a while before developers started to point out that, perhaps, control groups were not the right solution to this problem; it seems that they abstracted things a little too much. 

There were a few complaints about the control-group interface, but by far the loudest was that it failed to reflect the fact that CAT works on a per-CPU basis — each processor has its own set of CLOSIDs and its own active policy at any given time. The proposed interface was tied to processes rather than processors, and it forced the use of a single policy across the entire system. There are plenty of real-world use cases that want to have different cache-utilization policies on different CPUs in the same system, but the control-group mechanism could not express those policies. This problem was exacerbated by the fact that the number of CLOSIDs is severely limited; making it impossible for each CPU to use its own CLOSID-to-CBM mappings made that limitation much more painful. 

Beyond setting up different policies on different CPUs, many users would like to use the CPU as the primary determinant for cache policy. For example, a specific CPU running an important task could be given exclusive access to a large portion of the cache. If the task in question is bound to that processor, it will automatically get access to that cache reservation; any related processes — kernel threads performing work related to that task, for example — will also be able to use that cache space. This mode, too, is not well supported by an interface based on control groups. In its absence, users would have to track down each helper process and manually add it to the correct control group, a tedious and error-prone task. 

The problem was discussed repeatedly as new versions of the patch set came out during much of early 2015. At one point, Marcelo Tosatti posted [an interface based on `ioctl()` calls](<https://lwn.net/Articles/694805/>) that was meant to address some of the concerns, but it seems there was little interest in bringing `ioctl()` into the mix. In November, Thomas Gleixner posted [a description of how he thought the interface should work](<https://lwn.net/Articles/694806/>) for discussion. He said that a single, system-wide configuration was not workable and that ""we need to expose this very close to the hardware implementation as there are really no abstractions which allow us to express the various bitmap combinations"" His overall suggestion was to create a new virtual filesystem for the control of the CAT mechanism; that is the approach taken by Yu's current patch set. 

#### Herding the CAT

Returning to Yu's talk: he noted that [a new patch set](<https://lwn.net/Articles/694387/>) had been posted just prior to the conference; it shows the implementation of the new control interface. It is all based on a virtual filesystem, as Gleixner had suggested. Naturally enough, the name of that filesystem (`/sys/fs/rscctrl`) became the first topic of debate, with Gleixner [complaining](<https://lwn.net/Articles/694867/>) that it was too cryptic. Tony Luck's [suggestion](<https://lwn.net/Articles/694869/>) that it could instead be called: 

```
/sys/fs/Intel(R) Resource Director Technology(TM)/
```

seems unlikely to be adopted; "`/sys/fs/resctrl`" may emerge as the most acceptable name in the end. 

The top level of this filesystem contains three files: `tasks`, `cpus`, and `schemas`. The `tasks` file contains a list of all processes whose cache access is controlled by the bitmap found in the `schemas` file; similarly, `cpus` can be used to attach a bitmap to one or more CPUs. Initially the `tasks` file holds the IDs for all processes in the system, and `cpus` is all zeroes; the `schemas` file contains all ones. The default policy, thus, is to allow all processes in the system the full use of the cache. 

Normal usage will involve the system administrator creating subdirectories to create new policies; each subdirectory will contain the same set of three files. A different CBM can be written to the `schemas` file in the subdirectory, changing the cache-access policy for any affected process. A process can be tied to that new policy by writing its ID to the `tasks` file. It is also possible to tie the policy to one or more CPUs by writing a CPU mask to the `cpus` file. A CPU-based policy will override a process-ID-based one — if a process is running on a CPU with a specific policy, that is the policy that will be used regardless of whether the process has been explicitly set to use a different one. 

Yu's talk glossed over a number of details on exactly how these control files work, as one might expect; [the documentation file](<https://lwn.net/Articles/694870/>) from the patch set contains those details and some usage examples as well. He did discuss some benchmark results (which can be seen at the end of [his slides [PDF]](<https://events.linuxfoundation.org/sites/events/files/slides/cat8.pdf>)) showing significant improvements for specific workloads that were affected by heavy cache contention. This feature may not be needed by everybody, but it seems that some users will have a lot to gain from it. Realtime workloads, in particular, would appear to stand to benefit from dedicated cache space. 

As for where things stand: the current patch set is out for review, with the hope that the most significant obstacles have been overcome at this point. Assuming that the user-space interface issues have now been resolved, this code, which has been under development for well over a year, should be getting close to being ready for merging into the mainline. 

[Your editor would like to thank the Linux Foundation for supporting his travel to LinuxCon Japan].
