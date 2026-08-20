---
title: "KS2012: memcg/mm: Shared-memory accounting in memory cgroups"
url: https://lwn.net/Articles/516541/
date: "September 17, 2012"
category: "Memory management-Control groups"
author: "By Michael Kerrisk September 17, 2012 2012 Kernel Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Michael Kerrisk**  
September 17, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

During the 2012 Kernel Summit [memcg/mm minisummit](<https://lwn.net/Articles/516439/>), Greg Thelen and Michel Lespinasse led a session on improving shared-memory accounting for memory cgroups. Currently, memory cgroups memory usage accounting is done by charging each page to the first user to touch each page. In practice, this is usually okay, but it is potentially unfair when that page is shared. The process that faults the page could exit, but the cgroup is still charged for the page. Greg described another example where a pair of processes that are implemented as a producer/consumer, with each process running in separate cgroups, may be unfairly charged if they share the data using shared memory. His idea is that both producer and consumer have to touch the memory, and they should be charged equally for the resource. With the current implementation, the process that is charged is the one that first touches the memory--i.e., the producer. He would like to do something about this problem. 

One suggestion was to extend the `madvise()` system call to allow a process to claim ownership of any pages currently mapped in a specified range, but this would solve only some specific subsets of the problem. It may also be required that all future charges in an address range get charged to a specific cgroup, although it was unclear how this could be specified in a sensible fashion. Minimally, using `madvise()` would assume that the data was `mmap()`ed. Greg was asked to post a patch series for comment, and discussion will continue on the mailing list. 

[Next: NUMA scheduling](<https://lwn.net/Articles/516542/>)
