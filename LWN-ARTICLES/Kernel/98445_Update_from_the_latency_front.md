---
title: Update from the latency front
url: https://lwn.net/Articles/98445/
date: "August 18, 2004"
category: Latency; Voluntary preemption; proclatency
author: 
---

Efforts to track down and eliminate sources of latency in the 2.6 kernel continue. It seems, however, that most of the low-hanging fruit has been found; with the current iteration of the voluntary preemption patch, the remaining problems are rare and relatively hard to track down. So Ingo Molnar [built himself a new tool](<https://lwn.net/Articles/97811/>) to help with those harder cases. 

Ingo's problem with the previous preempt timing patch was that, while it showed where a lengthy latency took place, it yielded little information about what was happening during the high-latency event. So he adapted the profiling mechanism to bring a little light to the situation. With the latency tracing option turned on, a little tracing function gets called as part of every kernel function call. This trace code stores the time of the call into a large (4000 entries), per-CPU array. If the kernel avoids scheduling for too long, that array of function call information gets copied into a static array which is made available via `/proc/latency`. 

Ingo included some example output with his patch: 

```
preemption latency trace v1.0
      -----------------------------
       latency: 121 us, entries: 1032 (1032)
       process: default.hotplug/1470, uid: 0
       nice: -10, policy: 0, rt_priority: 0
      =======>
       0.000ms (+0.000ms): page_address (kmap_high)
       0.000ms (+0.000ms): page_slot (page_address)
       0.000ms (+0.000ms): flush_all_zero_pkmaps (kmap_high)
       0.000ms (+0.000ms): set_page_address (flush_all_zero_pkmaps)
      [...]
       0.118ms (+0.000ms): page_slot (set_page_address)
       0.118ms (+0.000ms): check_preempt_timing (sub_preempt_count)
```

The output shows the function call, and, in parentheses, the caller of each function. In this case, the output identifies `flush_all_zero_pkmaps()` as the real villain. 

Other changes to this patch include making hardware and software interrupts (which have been redirected into kernel threads) preemptible by default (""I reviewed a number of softirq users and it appears to be safe""), and, of course, the breaking up of more code which holds locks for too long.
