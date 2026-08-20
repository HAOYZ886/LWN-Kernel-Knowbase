---
title: "KS2012: memcg/mm: NUMA scheduling"
url: https://lwn.net/Articles/516542/
date: "September 17, 2012"
category: "Memory management-NUMA systems"
author: "By Michael Kerrisk September 17, 2012 2012 Kernel Summit"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Michael Kerrisk**  
September 17, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

The subject of the final session of the 2012 Kernel Summit [memcg/mm minisummit](<https://lwn.net/Articles/516439/>) was two patch sets that implement NUMA scheduling. The approaches taken by these patch sets have been described in previous LWN articles, one of which describes [Peter Zijlstra's approach](<https://lwn.net/Articles/486858/>), while the other describes [Andrea Arcangeli's approach (AutoNUMA)](<https://lwn.net/Articles/488709/>). 

Rik van Riel asked Peter and Andrea what changes would have to happen to the other's patch sets before they would be considered for merging. Only Andrea's AutoNUMA patch set was discussed in the allotted time. Peter reiterated some of the problems he discovered during review. Some of the problems are specific to the implementation and could be fixed, he thought, but he saw some of them as potentially fundamental design problems. Among the points that Peter noted were the following: 

  * The implementation must maintain fairness in the load balancer. This is a major sticking point, although there was some confusion as to whether Peter's concerns have already been addressed in the most recent version of the patch set or not. 

  * He strongly resists the idea that all tasks should be required to maintain statistics on their NUMA usage. Some long-lived processes will simply not care about their NUMA usage, but they will still incur the cost of maintaining the statistics. 

  * He strongly dislikes the per-page additional data that is required by Andrea's approach. 

  * Andrea's patches implement AutoNUMA on 32-bit systems. Peter believes such support is overkill. 

  * The overall emergent behavior of the AutoNUMA approach is difficult to understand and the way it is presented makes it difficult to predict how it will behave in practice. The full implementation is presented as a finished product; seeing the intermediate steps would have helped understanding the possible behavior. 

Peter and Andrea disagree on when the actual migration of a page from one node to another should take place. The fundamental disagreement hinges on whether the migration should happen asynchronously via some kernel thread, or whether the page should be unmapped and migrated on fault. Asynchronous migration can end up in the situation where a page migrates continually between nodes. Migrating on fault avoids the problem where pages ping-pong between nodes, but incurs a latency. Unfortunately, the path forward was not agreed upon, although it was said that it would be interesting to see both patch sets compared for varying sizes of machine and see at what point the complexity of Andrea's AutoNUMA approach is justified.
