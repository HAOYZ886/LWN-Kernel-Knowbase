---
title: NUMA placement problems
url: https://lwn.net/Articles/591995/
date: "March 26, 2014"
category: "Memory management-NUMA systems; NUMA"
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

The kernel's handling of task and memory placement has been the subject of a lot of discussion and development in recent years. The pace has slowed for the last few development cycles, but there is still work to be done in this area as can be seen by the discussion on the topic that was held at the 2014 Linux Storage, Filesystem, and Memory Management Summit. The session, led by Rik van Riel, Peter Zijlstra, and Mel Gorman, started with the question: lots of code has been merged, what should happen next? Peter made the observation that, while the code is in the kernel, few people have actually tried to take advantage of it to improve NUMA performance. What is most needed now is user feedback on how things are working and what could be improved. 

Davidlohr Bueso said that, on his systems, he can still get much better performance from a carefully hand-tuned configuration than with the automatic NUMA placement code. Rik added that, as far as he can tell, things work close to optimally on four-node systems, but tend to fall apart on systems with more nodes than that. Mel asked why that might be; there was some speculation that the costs of page hinting (tracking who is using each page of memory so that it can be moved to the right node) might be responsible, or perhaps the more complex topology of larger NUMA systems is not being handled well. But it seems that nobody really knows what the problems are. 

Mel said that truly understanding NUMA performance issues requires the collection of a lot of data. But that data collection is expensive, to the point that it disrupts the workload under study. It's hard enough for him to run his tests; he hasn't really found a good way for others to do it yet. It seems that Rik, Peter, and Mel each have their own way of measuring NUMA performance; they haven't done much talking among themselves in this area. That is, it was suggested, actually a good thing; each developer is able to find different problems with his particular approach. 

Rik noted that, while the NUMA code tries hard to keep anonymous pages on the same node as the processes using them, the same care is not yet applied to page cache pages. His question was: should it be? It is not clear that localization of the page cache would lead to better performance overall; Mel said that this area is pretty much ignored for now. 

Johannes Weiner said that a node-local page cache allocation policy might not make sense. If the system tries hard to allocate those pages locally, it could do so at the expense of pushing other useful pages out. At that point, the kernel is buying local pages at the expense of forcing disk I/O for other needed pages — probably not a good bargain. Currently, page reclaim is strongly tied to nodes, so some nodes can reclaim heavily while old pages languish on others. So, he said, it might be good to force some page aging on all nodes even if there isn't memory pressure everywhere. Then interleaving page cache pages across all nodes might be able to increase memory utilization and reduce the aging of useful pages, a win even if it results in more cross-node traffic. 

There were complaints that processes communicating over network sockets should be grouped onto the same node, but that doesn't happen now. There is seemingly a bit of a disconnect with the networking developers on how that kind of grouping should be done. There would also be value in moving network-oriented processes onto the NUMA node that holds the network adapter they are using, but there is no I/O awareness in the NUMA code at all currently. Improving the integration of networking and NUMA placement is not going to be an easy task; it will likely involve carrying NUMA information through many layers of the network stack. 

The session wound down without a lot in the way of hard conclusions. It seems clear that there is still a lot of work to be done in the area of NUMA placement. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
