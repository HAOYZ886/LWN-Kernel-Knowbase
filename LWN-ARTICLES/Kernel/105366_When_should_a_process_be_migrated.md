---
title: "When should a process be migrated?"
url: https://lwn.net/Articles/105366/
date: "October 6, 2004"
category: Scheduler
author: 
---

The performance of modern computers is heavily influenced by how well they use the processor's memory cache. Going to main memory is a slow operation (from a processor's point of view); an operating system which forces main memory accesses too often will run slowly. One of the things the Linux kernel does to optimize cache use is to try to avoid moving processes between CPUs if it is likely that those processes have a fair amount of useful data in the cache. When a process moves, it leaves its cached data behind and must begin populating the new CPU's cache from the beginning. That repopulation requires memory accesses and slows things down. 

The metric used by the kernel to decide whether moving a particular task is advisable is a scheduling domain parameter called `cache_hot_time`. If the process has run in the current processor within the "hot time," it is considered to have significant data in the cache and is not moved unnecessarily. In recent kernels, `cache_hot_time` for processors on non-NUMA, SMP systems is 2.5ms. 

Kenneth Chen recently [did some tests](<https://lwn.net/Articles/105368/>) to see if that value makes sense. On his four-processor system, he found that workload throughput with a 2.5ms hot time was 12% below its peak level - which happens with a 10ms value. As it turns out, 10ms was once the default value for the cache hot time; Kenneth proposes that this value be restored. Others have, instead, suggested that a new tunable parameter be provided so that administrators could find and set the optimal value for their systems. 

Ingo Molnar has come up with [a different approach](<https://lwn.net/Articles/105352/>) \- have the computer figure out for itself what the optimal "cache hot" time is. To this end, his code performs the following steps for each pair of processors on the system: 

  1. The first processor fills a large, shared buffer with data, thus populating its own cache with (some of) the contents of that buffer. 

  2. The second processor fills a private buffer, filling its own cache. 

  3. The second processor then overwrites the shared buffer, moving the contents of that buffer into its own cache. 

The time required for the third step is, to an approximation, a worst case scenario for what it costs to move a process when it has filled the local cache with data. Ingo tested the code on a few systems and got optimal values which vary from 5ms (on a four-processor Pentium 4 system) to 87ms (for an eight-processor, semi-NUMA, Pentium 3 system). Clearly, one default value for all systems is not the right answer. This also looks like a good number for the computer to find for itself - assuming subsequent tests show that this patch (or a successor) is finding something close to the optimal value.
