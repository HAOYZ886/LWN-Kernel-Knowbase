---
title: "KS2012: memcg/mm: Improving memory cgroups performance for non-users"
url: https://lwn.net/Articles/516533/
date: "September 17, 2012"
category: "Memory management-Control groups"
author: "By Michael Kerrisk September 17, 2012 2012 Kernel Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Michael Kerrisk**  
September 17, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

One of the criticisms of memory cgroups is that the implementation has some performance problems. During the 2012 Kernel Summit [memcg/mm minisummit](<https://lwn.net/Articles/516439/>), Mel Gorman made the point that the overhead of memory cgroups is very high, even when memory cgroups are not in use (i.e., the memory controller is configured in when the kernel is compiled and is enabled at run time, but there are no cgroups created, so every process is a member for the root cgroup). This was not the first time this issue had been noted, since Mel brought it up briefly at the April 2012 LSF/MM meeting, but this time he had performance profiles that were generated just prior to the minisummit. They had not been posted to any mailing list, but he mentioned that the [MMTests](<https://lwn.net/Articles/515285/>) configuration file to reproduce the profiles is trivial. One profile showed that 6% of the time for a page-fault microbenchmark was spent in `memcontrol.c` (i.e., broadly speaking, the code that implements memory cgroups). Another benchmark showed that 15% of the time was spent in `memcontrol.c`. Andi Kleen hypothesized that this is likely a cache-conflict issue. 

The overhead shown by these benchmarks is considered too great for a feature that is not in active use. Avoiding the problem by disabling memory cgroups with a kernel command-line parameter (or building the kernel with `CONFIG_MEMCG` disabled) wasn't considered to be an adequate response to the problem. There were many discussions on how the problem might be fixed. Glauber Costa suggested that the kernel should not account for cgroups until the first memory cgroup is created. At that point, it would be necessary to copy data from the core memory-management statistics; some synchronization would be necessary, but it should work. Further work would still be necessary to improve the performance when memory cgroups are enabled (though, probably there will always be some minimum overhead, even when the memory cgroups feature is not being actively used), but reducing the overhead when cgroups are not in use is the immediate problem. 

[Next: Memory-management performance topics](<https://lwn.net/Articles/516534/>)
