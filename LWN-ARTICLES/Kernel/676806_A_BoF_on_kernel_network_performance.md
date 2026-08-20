---
title: A BoF on kernel network performance
url: https://lwn.net/Articles/676806/
date: "February 24, 2016"
category: Networking
author: "By Nathan Willis February 24, 2016 Netdev/Netconf"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Nathan Willis**  
February 24, 2016

* * *

[Netdev/Netconf](<https://lwn.net/Archives/ConferenceByYear/#2016-Netdev>)

Whether one measures by attendance or by audience participation, one of the most popular sessions at the [Netdev 1.1](<http://www.netdevconf.org/1.1/>) conference in Seville, Spain was the network-performance birds-of-a-feather (BoF) session led by Jesper Brouer. The session was held in the largest conference room to a nearly packed house. Brouer and seven other presenters took the stage, taking turns presenting topics related to finding and removing bottlenecks in the kernel's packet-processing pipeline; on each topic, the audience weighed in with opinions and, often, proposed fixes. 

The BoF was not designed to produce final solutions, but rather to encourage debate and discussion--hopefully fostering further work. Debate was certainly encouraged, to the point where Brouer was not able to get to every topic on the agenda before time had elapsed. But what was covered provides a good snapshot of where network-optimization efforts stand today. 

#### DDoS mitigation

The first to speak was Gilberto Bertin from web-hosting provider CloudFlare. The company periodically encounters network bottlenecks on its Linux hosts, he said, with the most egregious being those caused by distributed denial-of-service (DDoS) attacks. Even a relatively small packet flood, such as two million UDP packets per second (2Mpps), will max out the kernel's packet-processing capabilities, saturating the receive queue faster than it can be emptied and causing the system to drop packets. 2Mpps is nowhere near the full 10G Ethernet wire speed of 14Mpps. 

DDoS attacks are usually primitive, and an `iptables` drop rule targeting each source address should suffice, but CloudFlare has found it insufficient. Instead, the company is forced to offload traffic to a user-space packet handler. Bertin proposed two approaches to solving the problem: using Berkeley Packet Filter (BPF) programs shortly after packet ingress to parse incoming packets (dropping DDoS packets before they enter the receive queue), and using circular buffers to process incoming traffic (thus eliminating many memory allocations). 

Brouer reported that he had tested several possible solutions himself, including using receive packet steering (RPS) and dedicating a CPU to handling the receive queue. Using RPS alone, he was able to handle 7Mpps in laboratory tests; by also binding a CPU, the number rose to 9Mpps. Audience members proposed several other approaches; Jesse Brandeburg suggested designating a queue for DDoS processing and steering other traffic away from it. Brouer discussed some tests he had run attempting to put drop rules as early as possible in the pipeline; none made a drastic difference in the throughput. When an audience member asked if BPF programs could be added to the network interface card's (NIC's) driver, David Miller suggested that running drop-only rules against the NIC's DMA buffer would be the fastest the kernel could possibly respond. 

There was also a lengthy discussion about how to reduce the overhead caused by memory operations. Brouer reported that `memcpy()` calls accounted to as much as 43% of the time required to process a received packet. Jamal Hadi Salim asked whether `sk_buff` buffers could simply be recycled in a ring; Alexander Duyck replied that not all NIC drivers would support that approach. Ultimately, Brouer wrapped up the topic by saying there was no clear solution: latency hides in a number places in the pipeline, so reducing cache misses, using bulk memory allocation, and re-examining the entire allocation strategy on the receive side may be required. 

#### Transmit powers

Brouer then presented the next topic, improving transmit performance. He noted that [bulk transmission](<https://lwn.net/Articles/615238/>) with the `xmit_more` API had solved the outgoing-traffic bottleneck, enabling the kernel to transmit packets at essentially full wire speed. But, he said, the "full wire speed" numbers are really achievable only in artificial workloads. For practical usage, it is hard to activate the bulk dequeuing discipline. Since the technique lowers CPU utilization, it would be beneficial to many users if it could be enabled well before one approaches the bandwidth limit. 

He suggested several possible alternative means to activate `xmit_more`, including setting off a trigger whenever the hardware transmit queue gets full, tuning Byte Queue Limits (BQLs), and providing a user-space API to activate bulk sending. He had experimented some with the BQL idea, he reported: adjusting the BQL downward until the bulk queuing discipline kicks in resulted in a 64% increase in throughput. 

Tom Herbert was not thrilled about that approach, noting that BQL was, by design, intended to be configuration-free; using it as a tunable feature seems like asking for trouble. John Fastabend asked if a busy driver could drop packets rather than queuing them, thus triggering the bulk discipline. Another audience member proposed adding an API through which the kernel could tell a NIC driver to split its queues. There was little agreement on approaches, although most in attendance seemed to feel that further discussion in this area was well warranted. 

#### The trials of small devices

Next, Felix Fietkau of the OpenWrt project spoke, raising concerns that recent development efforts in the kernel networking space focused too much on optimizing behavior for high-end Intel-powered machines, at the risk of hurting performance on low-end devices like home routers and ARM systems. In particular, he pointed out that these smaller devices have significantly smaller data cache sizes, comparable instruction cache sizes but without smart pre-fetchers, and smaller cache-line sizes. Some of the recent optimizations, particularly cache-line optimizations, can hurt performance on small systems, he said. 

He showed some benchmarks of kernel 4.4 running on a 720MHz Qualcomm QCA9558 system-on-chip. Base routing throughput was around 268Mbps; activating `nf_conntrack_rtcache` raised throughput to 360Mbps. Also removing `iptable_mangle` and `iptable_raw` increased throughput to 400Mbps. The takeaway, he said, was that removing or conditionally disabling unnecessary hooks (such as statistics-gathering hooks) was vital, as was eliminating redundant accesses to packet data. 

Miller commented that the transactional overhead of the hooks in question was the real culprit, and asked whether or not many of the small devices in question would be a good fit for hardware offloading via the switchdev driver model. Fietkau replied that many of the devices do support offload, but that it is usually crippled in some fashion, such as not being configurable. 

Fietkau also presented some out-of-tree hacks used to improve performance on small devices, including using lightweight socket buffers and using dirty pointer tricks to avoid invalidating the data cache. 

#### Caching

Brouer then moved on to the topic of instruction-cache optimization. The network stack, he said, does a poor job of utilizing the instruction cache, since the typical cache size is shorter than the code used to process the average Ethernet packet. Furthermore, even though many packets appearing in the same time window get handled in the same manner, processing each packet individually means each packet hits the same instruction-cache misses. 

He proposed several possible ways to better utilize the cache, starting with processing packets in bundles, enabling several to be processed simultaneously at each stage. NIC drivers could bundle received packets, he said, for more optimal processing. The polling routine already processes many packets at once, but it currently calls "the full stack" for each packet. And the driver can view all of the packets available in the receive ring, so it could simply treat them all as having arrived at the same time and process them in bulk. A side effect of this approach, he said, would be that it hides latency caused by cache misses. 

A related issue, he said, is that the first cache miss often happens too soon for prefetching, in the `eth_type_trans()` function. By delaying the call to `eth_type_trans()` in the network stack's receive loop, the miss can be avoided. Even better, he said, would be to avoid calling `eth_type_trans()` altogether. The function is used to determine the packet's protocol ID, he said, which could also be determined from the hardware RX descriptor. 

Brouer also proposed staging bundles of packets for processing at the generic receive offload (GRO) and RPS layers. GRO does this already, he said, though it could be further optimized. Implementing staged processing for RPS faces one hurdle in the fact that RPS takes cross-CPU locks for each packet. But Eric Dumazet pointed out that bulk enqueuing for remote CPUs should be easily doable. RPS already defers sending the inter-processor interrupt, which essentially amortizes the cost across multiple packets. 

#### TC and other topics

Fastabend then spoke briefly (as time was running short) about the queuing discipline (qdisc) code path in the kernel's traffic control (TC) mechanism. Currently, qdisc takes six lock operations, even if the queue is empty and the packet is transmitted directly. He ran some benchmarks that showed that the locks account for 70-82% of the time spent in qdisc, and thus set out to re-implement qdisc in a lockless manner. He has posted an RFC implementation that reduces the lock count to two; the work is, therefore, not complete yet, but there are other items remaining on the to-do list. One is support for bulk dequeuing, the other is gathering some real-world numbers to determine if the performance improvement is as anticipated. 

Brouer then gave a quick overview of the "packet-page" concept: at a very early point in the receive process, a packet could be extracted from the receive ring into a memory page, allowing it to be sent on an alternative processing path. "It's a crazy idea," he warned the crowd, but it has several potential use cases. First, it could be a point for kernel-bypass tools (such as the [Data Plane Development Kit](<http://dpdk.org/>)) to hook into. It could also allow the outgoing network interface to simply move the packet directly into the transmit ring, and it could be useful for virtualization (allowing guest operating systems to rapidly forward traffic on the same host). Currently, implementing packet-page requires hardware support (in particular, hardware that marks packet types in the RX descriptor), but Brouer reported that he has seen some substantial and encouraging results in his own experiments. 

As the session time finally elapsed for good, Brouer also briefly addressed some ideas for reworking the memory-allocation strategy for received packets (as alluded to in the first mini-presentation of the BoF). One idea is to write a new allocator specific to the network receive stack. There are a number of allocations identified as introducing overhead, so there is plenty of room for improvement. 

But other approaches are possible, too, he said. Perhaps using a DMA mapping would be preferable, thus avoiding all allocations. There are clear pitfalls, such as needing a full page for each packet and the overhead of clearing out enough headroom for inserting each `sk_buff`. 

Finally, Brouer reminded the audience of just how far the kernel networking stack has come in recent years. In the past two years alone, he said, the kernel has moved from a maximum transmit throughput of 4Mpps to the full wire speed of 14.8Mpps. IPv4 forwarding speed has increased from 1Mpps to 2Mpps on single core machines (and even more on multi-core machines). The receive throughput started at 6.4Mpps and, with the latest experimental patches, now hovers around 12Mpps. Those numbers should be an encouragement; if the BoF attendees are anything to judge by, further performance gains are no doubt on the horizon still. 

_[The author would like to thank the Netconf and Netdev organizers for travel assistance to Seville.]_
