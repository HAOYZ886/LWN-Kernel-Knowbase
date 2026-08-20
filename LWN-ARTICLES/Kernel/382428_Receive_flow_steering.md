---
title: Receive flow steering
url: https://lwn.net/Articles/382428/
date: "April 7, 2010"
category: Networking
author: "By Jake Edge April 7, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
April 7, 2010

Today's increasing bandwidth, and faster networking hardware, has made it difficult for a single CPU to keep up. Multiple cores and packages have helped matters on the transmit side, but the receive side is trickier. Tom Herbert's receive packet steering (RPS) patches, which we [looked](<http://lwn.net/Articles/362339/>) at back in November, provide a way to steer packets to particular CPUs based on a hash of the packet's protocol data. Those patches were applied to the network subsystem tree and are bound for 2.6.35, but now Herbert is back with an enhancement to RPS that will attempt to steer packets to the CPU on which the receiving application is running: [receive flow steering](<http://lwn.net/Articles/381955/>) (RFS). 

RFS uses the RPS hash table to store the CPU of an application when it calls `recvmsg()` or `sendmsg()`. Instead of picking an arbitrary CPU based on the hash and a CPU mask optionally set by an administrator, as RPS does, RFS tries to use the CPU where the receiving application is running. Based on the hash calculated on the incoming packet, RFS can look up the "proper" CPU and assign the packet there. 

The RPS CPU masks, which can be set via sysfs for each device (and queue for devices with multiple queues), represent the allowable CPUs to assign for a packet. But dynamically changing those values introduces the possibility of out-of-order packets. For RPS, with largely static CPU masks, it was not necessarily a big problem. For RFS, however, multiple threads trying to read from the same socket, while potentially bouncing around to different CPUs, would cause the CPU value in the hash table to change frequently, thus increasing the likelihood of out-of-order packets. 

For RFS, that was considered to be a ""non-starter"", Herbert said, so a different approach was required. To eliminate the out-of-order packets, two types of hash tables are created, both indexed by the hash calculated from the packet information. The global `rps_sock_flow_table` is populated by the `recvmsg()` or `sendmsg()` call with the CPU number where the application is running (this is called the "desired" CPU). Each device queue then gets a `rps_dev_flow_table` which contains the most recent CPU used to handle packets for that connection (which is called the "current" CPU). In addition, the value of the tail queue counter for the current CPU's backlog queue is stored in the `rps_dev_flow_table` entry. 

The two CPU values are compared when deciding which CPU to process the packet on (which is done in `get_rps_cpu()`). If the current CPU (as determined from the `rps_dev_flow_table` hash table) is unset (presumably for the first packet) or that CPU is offline, the desired CPU (from `rps_sock_flow_table`) is used. If the two CPU values are the same, obviously, that CPU is used. But if they are both valid CPU numbers, but different, the backlog tail queue counter is consulted. 

Backlog queues have a queue head counter that gets incremented when packets are removed from the queue. Using that and the queue length, a queue tail counter value can be calculated. That is what gets stored in `rps_dev_flow_table`. When the kernel makes its decision about which CPU to assign the packet to, it needs to consider both the current (really last used by the kernel) CPU and the desired (last used by an application for sending or receiving) CPU. 

The kernel compares the current CPU's queue tail counter (as stored in the hash table) with that CPU's queue head counter. If the tail counter is less than or equal the head counter, that means that all packets that were put on the queue by this connection have been processed. That in turn means that switching to the desired CPU will not result in out-of-order packets. 

Herbert's current patch is for TCP, but RFS should be ""usable for other flow oriented protocols"". The benefit is that it can achieve better CPU locality for the processing of the packet, both by the kernel, and the application itself. Depending on various factors--cache hierarchy and application are given as examples--it can and does increase the packets per second that can be processed as well as lowering the latency before a packet gets processed. But, interestingly, ""on simple benchmarks, we don't necessarily see improvement and sometimes see degradation"". 

For more complex benchmarks, the performance increase looks to be significant. Herbert gave numbers for a netperf run where the transactions per second went from 104K without either RFS or RPS, to 290K for the best RPS configuration, and to 303K with RFS and RPS. A different test, with 100 threads handling an RPC-like request/response with some user-space work being done, was even more dramatic. That test showed 103K, 174K, and 223K respectively, but also showed a marked decrease in the latency for both RPS and RPS + RFS. 

These patches are coming from Google, which has been known to process a few packets using the Linux kernel. If RFS is being used on production systems at Google, that would seem to bode well for its reliability and performance beyond just benchmarks. The patches were posted April 2, and seemed to be generally well-received, so it's a little early to tell when they might make it into the mainline. But it seems rather likely that we will see them in either 2.6.35 or 36.
