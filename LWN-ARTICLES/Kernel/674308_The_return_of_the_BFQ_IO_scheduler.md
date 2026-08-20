---
title: The return of the BFQ I/O scheduler
url: https://lwn.net/Articles/674308/
date: "February 3, 2016"
category: "Block layer-IO scheduling"
author: "By Jonathan Corbet February 3, 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 3, 2016

Once upon a time, block I/O schedulers, which are charged with optimizing the stream of I/O requests to storage devices, were an area of intensive development. The characteristics of rotating drives meant that I/O performance would vary greatly depending on the order in which requests were serviced. Hardware changes (and the move to solid-state storage devices in particular) have taken away much of the motivation for work in this area; on modern drives, the best thing the I/O scheduler can do is often to just stay out of the way. Still, some interest in improving I/O schedulers remains, as can be seen in [the return of the budget fair queuing (BFQ) scheduler](<https://lwn.net/Articles/674300/>). 

BFQ, developed by Paolo Valente and Arianna Avanzini, was last [covered here](<https://lwn.net/Articles/601799/>) in June 2014. It is derived from the CFQ I/O scheduler found in current kernels; CFQ is the default for most systems, though some users replace it with one of the simpler schedulers when they have fast drives. BFQ has diverged a long way from its CFQ roots, though. 

See the 2014 article for details on how BFQ works; what follows is a much-abbreviated summary. For any given block device, BFQ creates a request queue for each process in the system. Each queue is assigned a budget, being the amount of I/O traffic it is allowed to dispatch to the drive in any given scheduling cycle; the budget is determined by the scheduler itself based on I/O priority and observations of the process's I/O behavior. The dispatching code, given the mellifluous name "B-WF2Q+", services each queue in turn; an amount of I/O up to the associated budget will be dispatched during that turn. By setting the budgets properly, BFQ tries to ensure that all processes get a fair amount of the available I/O bandwidth within fairly tight latency limits. 

The core idea is simple, but the real world is rather more complex; as a result, BFQ, like CFQ before it, has accrued a number of heuristics aimed at improving performance in specific areas. Queues for cooperating processes working in the same area of the disk are merged to enable more request consolidation. If a process doing read requests empties its queue before exhausting its budget, the device will be idled briefly to give that process a chance to create another request. Processes identified as "soft realtime" (those with moderate bandwidth requirements and an observed regular on/off request cycle) will get higher priority, as will processes that are just starting up. And so on. 

By all appearances, BFQ has not changed a great deal since the June 2014 posting. The described heuristics are mostly the same. Undoubtedly bugs have been fixed and performance improved based on experience using the scheduler in the real world. Such experience does exist; the scheduler has been shipped with a few second-tier distributions and, evidently, has been used in some handsets. Paolo has previously posted from the [University of Modena and Reggio Emilia](<http://unimore.it/>), but the current patches come instead from a Linaro address, suggesting that there is some commercial interest in this work. 

The BFQ scheduler was well received in 2014, but the proposed method of getting it into the kernel was not. At that time, BFQ was added as a new I/O scheduler alongside the others in the kernel, but the kernel community had little appetite for yet another scheduler, much less one that resembles the in-kernel CFQ scheduler so closely. Since BFQ is, by most appearances, an improvement on CFQ, Paolo was advised to generate a series of patches evolving CFQ in the desired direction. That was easier said than done, even given that BFQ derives from CFQ; the two schedulers had diverged considerably while BFQ was being developed. As a result of this requirement, BFQ essentially disappeared from the kernel mailing lists for more than a year. 

Its return shows how the BFQ developers intend to try to satisfy the request that was made of them. The new patch set consists of 22 changesets, divided into three main phases: 

  1. The CFQ scheduler is regressed back to the state it was in when BFQ originally split off from it. The first eight patches simply strip features (mostly heuristics) out of CFQ, simplifying it considerably. At the end of this phase, CFQ remains and (presumably) still works, but lacks most of the features it has acquired in the last few years. 

  2. The core CFQ engine is replaced wholesale with the core BFQ engine; that patch removes 1,700 lines of code and adds nearly 4,300 lines. The scheduler still calls itself CFQ (a name that may never change for compatibility reasons); at this point it represents BFQ in a relatively early stage of development. The next patch adds back full hierarchical scheduling and control-group support. 

  3. The remaining 11 patches add new heuristics to BFQ, addressing various performance and fairness issues that have come up over time. 

As of this writing, the new patch set has not yet received any comments, so it remains to be seen whether the development community will accept this approach or not. As a general rule, kernel developers like to see code evolved in relatively small steps; a massive replacement of code in a single patch is hard to review and has a relatively high chance of introducing regressions and performance problems. The CFQ scheduler is heavily used in production; destabilizing it for a couple of releases is not really a viable option. So it's entirely possible that this submission will be no more successful than the previous ones. 

On the other hand, there may be no better way to get BFQ into the kernel at this point. If enthusiasm for BFQ were low, that might simply doom it to an out-of-tree existence. But BFQ seems demonstrably better than the CFQ scheduler, and its heuristics are better understood, so there is a real motivation to get it into the kernel. One can imagine that it might take some time to build a sufficiently high level of confidence in the quality of the code, so BFQ should not be expected in, say, the 4.6 development cycle. But, given that time, the development community might yet see a way to get this code merged and made available to the user community as a whole.
