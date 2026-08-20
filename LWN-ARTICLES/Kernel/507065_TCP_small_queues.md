---
title: TCP small queues
url: https://lwn.net/Articles/507065/
date: "July 17, 2012"
category: "Networking-Bufferbloat"
author: "By Jonathan Corbet July 17, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 17, 2012

The "bufferbloat" problem is the result of excessive buffering in the network stack; it leads to long latencies and poor reliability in the network as a whole. Fixing it is a matter of buffering less data in each system between any two endpoints—a task that sounds simple, but proves to be more challenging than one might expect. It turns out that buffering can show up in many surprising places in the networking stack; tracking all of these places down and fixing them is not always easy. 

A number of bloat-fighting changes have gone into the kernel over the last year. The [CoDel queue management algorithm](<https://lwn.net/Articles/496509/>) works to prevent packets from building up in router queues over time. At a much lower level, [byte queue limits](<https://lwn.net/Articles/454390/>) put a cap on the amount of data that can be waiting to go out a specific network interface. Byte queue limits work only at the device queue level, though, while the networking stack has other places—such as the queueing discipline level—where buffering can happen. So there would be value in an implementation that could limit buffering at levels above the device queue. 

Eric Dumazet's [TCP small queues patch](<https://lwn.net/Articles/506237/>) looks like it should be able to fill at least part of that gap. It limits the amount of data that can be queued for transmission by any given socket regardless of where the data is queued, so it shouldn't be fooled by buffers lurking in the queueing, traffic control, or netfilter code. That limit is set by a new sysctl knob found at: 

```
/proc/sys/net/ipv4/tcp_limit_output_bytes
```

The default value of this limit is 128KB; it could be set lower on systems where latency is the primary concern. 

The networking stack already tracks the amount of data waiting to be transmitted through any given socket; that value lives in the `sk_wmem_alloc` field of `struct sock`. So applying a limit is relatively easy; `tcp_write_xmit()` need only look to see if `sk_wmem_alloc` is above the limit. If that is the case, the socket is marked as being throttled and no more packets are queued. 

The harder part is figuring out when some space opens up and it is possible to add more packets to the queue. The time when queue space becomes free is when a queued packet is freed. So Eric's patch overrides the normal `struct sk_buff` destructor when an output limit is in effect; the new destructor can check to see whether it is time to queue more data for the relevant socket. The only problem is that this destructor can be called from deep within the network stack with important locks already held, so it cannot queue new data directly. So Eric had to add a new tasklet to do the actual job of queuing new packets. 

It seems that the patch is having the intended result: 

Results on my dev machine (tg3 nic) are really impressive, using standard pfifo_fast, and with or without TSO/GSO. Without reduction of nominal bandwidth. I no longer have 3MBytes backlogged in qdisc by a single netperf session, and both side socket autotuning no longer use 4 Mbytes. 

He also [ran some tests](<https://lwn.net/Articles/507090/>) over a 10Gb link and was able to get full wire speed, even with a relatively small output limit. 

There are some outstanding questions, still. For example, Tom Herbert [asked](<https://lwn.net/Articles/507095/>) about how this mechanism interacts with more complex queuing disciplines; that question will take more time and experimentation to answer. Tom also suggested that the limit could be made dynamic and tied to the lower-level byte queue limits. Still, the patch seems like an obvious win, so it has already been pulled into the net-next tree for the 3.6 kernel. The details can be worked out later, and the feature can always be turned off by default if problems emerge during the 3.6 development cycle.
