---
title: The NET policy mechanism
url: https://lwn.net/Articles/696856/
date: "August 10, 2016"
category: Networking
author: "By Jonathan Corbet August 10, 2016"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 10, 2016

One of the heuristics that guide kernel development says that, whenever possible, the addition of tuning knobs should be resisted. Such knobs are seen as the developer giving up and pushing a tuning problem onto users; instead, the kernel should, whenever possible, tune itself to suit the current workload. An attempt to reduce the user's tuning responsibilities for the networking subsystem is running into resistance, though. 

Arguably, no part of the kernel offers more opportunities for user tuning than networking. Queuing disciplines and traffic control allow the creation of elaborate, in-kernel routing for packets. Interrupt affinities and device polling can be tweaked, there are numerous congestion-control algorithms to choose between, queue lengths and packet-ring sizes can be played with, and so on. There is also a whole set of policies and knobs that can be set within the network interfaces themselves. The result is a subsystem with a great deal of flexibility, but also one that is complex and difficult for most people to tune properly. Thus, many administrators do not even try if they can avoid it. Unfortunately, they often cannot avoid it; as Ken Liang noted in the introduction to his [kernel NET policy patch set](<https://lwn.net/Articles/696458/>), ""network performance is not good with default system settings."" 

That patch set introduces a new high-level policy mechanism; the administrator can use it to describe the sort of workload that the networking subsystem should be tuned for. The options are: 

  * **CPU** : the most important factor is reducing the amount of CPU time needed to keep up with the network. 

  * **Latency** : the latency of network communications should be kept to a minimum. 

  * **Throughput** : the goal is to push the maximum amount of data through the network. 

These policies may be set at a per-interface level, in which case they apply to all communications flowing through the affected interface. Policies can also be set on a per-task and per-socket level, though, allowing different users to operate under different policies. In this case, the interface-level policy must be set to the special "mixed" option; if the interface is given any other policy, all communications through that interface must match that policy. 

Exactly how these policies are implemented is not well documented in the patch set; that is not helped by the fact that, in the current version, there are no driver-level patches implementing the new policy-setting hooks. That support can be seen in [a previous version of the patch set](<https://lwn.net/Articles/694754/>); it was seemingly removed in response to complaints about the length of the series as a whole. Therein, one sees that much of the functionality is dependent on Intel's "Ethernet Flow Director" technology, though Liang maintains that it can be made to work on any adapter that supports loadable flow-direction rules — as many high-end adapters do. 

One aspect of the policy implementation is interrupt mitigation. Most high-speed network adapters can handle vast numbers (as in millions) of packets per second; if they generated interrupts for every packet sent or received, the system would be swamped. So these adapters support various mechanisms for reducing the number of interrupts delivered. This is where the policy comes in: reducing the number of interrupts raised by the interface can increase the amount of time it takes to process a packet, thus increasing latency. So a latency-sensitive policy will tolerate more interrupts, while a CPU-conserving policy will reduce interrupts to a minimum. 

Multi-queue devices (the only type supported by this patch set) can steer packets to specific queues and vary their interrupt behavior for each. Multiple queues can be used to support policy goals in other ways as well; throughput-oriented queues can be longer and run at lower priority, while latency-oriented queues should be high-priority and short. So the other aspect of the NET policy patches is queue-selection logic that depends on the policy attached to each packet. When a policy is established, the queues (and their CPU/interrupt affinities) are set up automatically, so the administrator need not deal with that sort of complexity. 

It will surprise few readers to learn that a number of networking developers expressed concerns about this patch set. Policy implementation in the kernel is generally something that developers try to avoid; the kernel is meant to implement mechanism, leaving policy decisions to others. Given that most of what the NET policy patches do can already be done from user space, some questioned why the remaining bits weren't added to the API so that policy selection could be done outside of the kernel. 

The answer to this question, as found in the cover letter to the series, goes something like this. User space does not have access to the same level of information that the kernel has, and the information that is available can be stale and subject to race conditions. If you do push these decisions out to user space, you'll add more context switches and slow down the system as a whole. And only the kernel can manage competing requests from multiple users in a way that's fair to all. The networking developers understand these arguments, but not everybody seems convinced that solving the problem in user space is impossible. 

Also, perhaps inevitably, it was suggested that, rather than coding queue selection into the policy code, that decision could be [made by an eBPF program](<https://lwn.net/Articles/696861/>) loaded from user space. Using eBPF would certainly add flexibility to the system, but it seems unlikely to make the task of policy administration easier. 

As things stand now, it seems clear that quite a bit more effort will be required to convince the network development community that the NET policy patches are the best solution to the problem. But the problem itself is real; as Stephen Hemminger [put it](<https://lwn.net/Articles/696862/>), ""network tuning is hard, most people get it wrong, and nobody agrees on the right answer."" Creating a set of canned policies in the kernel may not be the best solution to the problem, but the real proof of that would be to come up with a better solution, and those seem to be in short supply at the moment.
