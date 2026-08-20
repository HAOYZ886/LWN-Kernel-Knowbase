---
title: Improving Linux networking performance
url: https://lwn.net/Articles/629155/
date: "January 13, 2015"
category: "Networking-Performance"
author: "By Jonathan Corbet January 13, 2015 LCA 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 13, 2015

* * *

[LCA 2015](<https://lwn.net/Archives/ConferenceByYear/#2015-linux.conf.au>)

100Gb network adapters are coming, said Jesper Brouer in his talk at the LCA 2015 kernel miniconference ([slides [PDF]](<http://people.netfilter.org/hawk/presentations/LCA2015/net_stack_challenges_100G_LCA2015.pdf>)). Driving such adapters at their full wire speed is going to be a significant challenge for the Linux kernel; meeting that challenge is the subject of his current and future work. The good news is that Linux networking has gotten quite a bit faster as a result — even if there are still problems to be solved. 

#### The challenge

As network adapters get faster, the time between packets (i.e. the time the kernel has to process each packet) gets smaller. With current 10Gb adapters, there are 1,230ns between two 1538-byte packets. 40Gb networking cuts that time down significantly, to 307ns. Naturally, 100Gb exacerbates the problem, dropping the per-packet time to about 120ns; the interface, at this point, is processing 8.15 million packets per second. That does not leave a lot of time to figure out what to do with each packet. 

So what do you do if you, like almost all of us, do not have a 100Gb adapter around to play with? You use a 10Gb adapter with small frames instead. The smallest Ethernet frame that can be sent is 84 bytes; on a 10G adapter, Jesper said, there are 67.2ns between minimally-sized packets. A system that can cope with that kind load should be positioned to do [![\[Jesper Brouer\]](https://static.lwn.net/images/conf/2015/lca/JesperBrouer-sm.jpg)](<https://lwn.net/Articles/629276/>) something reasonable with 100Gb networking when it becomes available. But coping with that load is hard: on a 3GHz CPU, there are only about 200 CPU cycles available for the processing of each packet. That, Jesper noted, is not a lot. 

The kernel has traditionally not done a great job with this kind of network-intensive workload. That has led to existence of a number of out-of-tree networking implementations that bypass the kernel's network stack entirely. The demand for such systems indicates that the kernel is not using the hardware optimally; the out-of-tree implementations can drive adapters at full wire speed from a single CPU, which the mainline kernel is hard-put to do. 

The problem, Jesper said, is that the kernel developers have focused on scaling out to large numbers of cores. In the process, they have been able to hide regressions in per-core efficiency. The networking stack, as a result, works well for many workloads, but workloads that are especially latency-sensitive have suffered. The kernel, today, can only forward something between 1M and 2M packets per core every second, while some of the bypass alternatives approach a rate of 15M packets per core per second. 

#### Time budgets

If you are going to address this kind of problem, you have to take a hard look at the cost of every step in the processing of a packet. So, for example, a cache miss on Jesper's 3GHz processor takes about 32ns to resolve. It thus only takes two misses to wipe out the entire time budget for processing a packet. Given that a socket buffer ("SKB") occupies four cache lines on a 64-bit system and that much of the SKB is written during packet processing, the first part of the problem is apparent — four cache misses would consume far more than the time available. 

Beyond that, using the x86 `LOCK` prefix for atomic operations takes about 8.25ns. In practice, that means that the shortest spinlock lock/unlock cycle takes a little over 16ns. So there is not room for a lot of locking within the time budget. 

Then there is the cost of performing a system call. On a system with SELinux and auditing enabled, that cost is just over 75ns — over the time budget on its own. Disabling auditing and SELinux reduces the time required to just under 42ns, which is better, but that is still a big part of the time budget. There are ways of amortizing that cost over multiple packets; they include system calls like `sendmmsg()`, `recvmmsg()`, `sendfile()`, and `splice()`. In practice, he said, they do not work as well as he expected, but he did not get into why. From the audience, Christoph Lameter noted that latency-sensitive users tend to use the InfiniBand "IB verbs" mechanism. 

Given all of these costs, Jesper asked, how do the network-bypass solutions achieve higher performance? The key appears to be batching of operations, along with preallocation and prefetching of resources. These solutions keep work CPU-local and avoid locking. It is also important to shrink packet metadata and reduce the number of system calls. Faster, cache-optimal data structures also help. Of all of these techniques, batching of operations is the most important. A cost that is intolerable on a per-packet basis is easier to absorb if it is incurred once per dozens of packets. 16ns of locking per packet hurts; if sixteen packets are processed at once, that overhead drops to 1ns per packet. 

#### Improving batching

So, unsurprisingly, Jesper's work has been focused on improving batching in the networking layer. It includes the [TCP bulk transmission work](<https://lwn.net/Articles/615238/>) that was covered here in October; see that article for details on how it works. In short, it is a mechanism for informing network drivers that there are more packets waiting for transmission, allowing the driver to delay expensive operations until all of those packets have been queued. With this work in place, his system can transmit 14.8M packets per second — at least if it's the same little packet sent over and over again. 

The tricky part, he said, is adding batching APIs to the networking stack without increasing the latency of the system. Latency and throughput must often be traded off against each other; here the objective is to optimize both. An especially hard trick to resist is speculative transmission delays — a bet that another packet is coming soon. Such tricks tend to improve benchmark results but are less useful for real-world workloads. 

Batching can — and should — be done at multiple layers in the stack. So, for example, the queuing discipline ("qdisc") subsystem is a good place for batching; after all, delays are already happening as the result of queueing. In the best case, currently, the qdisc code requires six `LOCK` operations per packet — 48ns of pure locking overhead. The full cost of queuing a packet is 58-68ns, so the bulk of the time spent is with locking operations. Jesper has worked to add batching, spreading that cost over multiple packets, but that only works if there is actually a queue of packets. 

The nominal fast path through the qdisc code happens when there is no queue; in such situations, packets can often be passed directly to the network interface and not queued at all. Currently, such packets incur the cost of all six `LOCK` operations. It should, he said, be possible to do better. A lockless qdisc subsystem could eliminate almost all the cost of queuing packets. Jesper has a test implementation to demonstrate what can be done; eliminating a minimum of 48ns of overhead, he said, is well worth doing. 

While transmission performance now looks reasonably good, he said, receive processing can still do with some improvement. A highly tuned setup can receive a maximum of about 6.5M packets per second — and that's when the packets are simply being dropped after reception. Some work on optimizing the receive path is underway, raising that maximum to just over 9M packets per second. But there is a problem with this benchmark: it doesn't show the costs of interaction with the memory-management subsystem. 

#### Memory management

And that interaction, it turns out, is painful. The network stack's receive path, it seems, has some behavioral patterns that do not bring out the best behavior in the slab allocators. The receive code can allocate space for up to 64 packets at a time, while the transmit path can free packets in batches of up to 256. These pattern seems to put the SLUB allocator, in particular, into a relatively slow path. Jesper did some microbenchmarking and found that a single `kmem_cache_alloc()` call followed by `kmam_cache_free()` required about 19ns. But when 256 allocations and frees were done, that time increased to 40ns. In real-world use in the networking stack, though, where other things are being done as well, the allocation/free overhead grows even more, to 77ns — more than the time budget on its own. 

Thus, Jesper concluded, there need to be either improvements to the memory-management code or some way of bypassing it altogether. To try the latter approach, he implemented a subsystem called [qmempool](<https://lwn.net/Articles/625427/>); it does bulk allocation and free operations in a lockless manner. With qmempool, he was able to save 12ns in simple tests, and up to 40ns in packet forwarding tests. There are a number of techniques used in qmempool to make it faster, but the killer feature is the batching of operations. 

Jesper wound down by saying that qmempool was implemented as a sort of provocation: he wanted to show what was possible and inspire the memory-management developers to do something about it. The response from the memory-management camp was covered in the next talk, which will be [reported on separately](<https://lwn.net/Articles/629152/>). 

[Your editor would like to thank linux.conf.au for funding his travel to the event.]
