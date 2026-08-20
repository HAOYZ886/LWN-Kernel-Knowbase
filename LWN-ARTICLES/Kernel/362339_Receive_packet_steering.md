---
title: Receive packet steering
url: https://lwn.net/Articles/362339/
date: "November 17, 2009"
category: Networking
author: "By Jonathan Corbet November 17, 2009"
---

By **Jonathan Corbet**  
November 17, 2009

Contemporary networking hardware can move a lot of packets, to the point that the host computer can have a hard time keeping up. In recent years, CPU speeds have stopped increasing, but the number of CPU cores is growing. The implication is clear: if the networking stack is to be able to keep up with the hardware, smarter processing (such as [generic receive offload](<http://lwn.net/Articles/358910/>)) will not be enough; the system must also be able to distribute the work across multiple processors. Tom Herbert's [receive packet steering](<http://lwn.net/Articles/361440/>) (RPS) patch aims to help make that happen. 

From the operating system's point of view, distributing the work of outgoing data across CPUs is relatively straightforward. The processes generating data will naturally spread out across the system, so the networking stack does not need to think much about it, especially now that multiple transmit queues are supported. Incoming data is harder to distribute, though, because it is coming from a single source. Some network interfaces can help with the distribution of incoming packets; they have multiple receive queues and multiple interrupt lines. Others, though, are equipped with a single queue, meaning that the driver for that hardware must deal with all incoming packets in a single, serialized stream. Parallelizing such a stream requires some intelligence on the part of the host operating system. 

Tom's patch provides that intelligence by hooking into the receive path - `netif_rx()` and `netif_receive_skb()` \- right when the driver passes a packet into the networking subsystem. At that point, it creates a hash from the relevant protocol data (IP addresses and port numbers, in particular) and uses it to pick a CPU; the packet is then enqueued for the target CPU's attention. By default, any CPU on the system is fair game for network processing, but the list of target CPUs for any given interface can be configured explicitly by the administrator if need be. 

The code is relatively simple, but it succeeds in distributing the load of receive processing across the system. The use of the hash is important: it ensures that packets for the same stream of data end up on the same processor, increasing cache locality (and, thus, performance). This scheme is also nice in that it requires no driver changes at all, so it can be deployed quickly and with minimal disruption. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There is one place where drivers can help, though. The calculation of the hash requires accessing data from the packet header. That access will necessarily involve one or more cache misses on the CPU running the steering code - that data was just put there by the network interface and thus cannot be in any CPU's cache. Once the packet has been passed over to the CPU which will be doing the real work, that cache miss overhead is likely to be incurred again. Unnecessary cache misses are the bane of high-speed network processing; quite a bit of work has been done to eliminate them wherever possible. Adding a new cache miss for every packet in the steering code would be counterproductive. 

It turns out that a number of network interfaces can, themselves, calculate a hash value for incoming packets. That processing comes for free, and it could eliminate the need to calculate that hash (and suffer the overhead of accessing the data) on the dispatching processor. To take advantage of this capability, the RPS patch adds a new `rxhash` field to the `sk_buff` (SKB) structure. Drivers which are able to obtain hash values from the hardware can place them in the SKB; the network stack will then skip the calculation of its own hash value. That should keep the packet's data out of the dispatching CPU's cache entirely, speeding processing. 

How well does this work? The patch included some benchmark results using the `netperf` tool. An 8-core server with a tg3-based network interface went from 90,000 transactions per second to 285,000; an e1000-based adapter on the same system went from 90,000 to 292,000. Similar results are obtained for nForce and bnx2x chipsets on 16-core servers. It would appear that this patch does succeed in making networking processing faster on multi-core systems. 

The patch, incidentally, comes from Google, which has a bit of experience with network processing. It has, evidently, been running on Google's production servers for a while. So the RPS patch is, hopefully, an early component of what will be a broad stream of contributions from Google as that company tries to work more closely with the mainline. It seems like a good start.
