---
title: Hierarchical group I/O scheduling
url: https://lwn.net/Articles/427961/
date: "February 15, 2011"
category: "Block layer-IO scheduling"
author: "By Jonathan Corbet February 15, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
February 15, 2011

There has recently been much attention paid to the group CPU scheduling feature built into the Linux kernel. Using group scheduling, it is possible to ensure that some groups of processes get a fair share of the CPU without being crowded out by a rather larger number of CPU-intensive processes in a different group. Linux has supported this feature for some years, but it has languished in relative obscurity; it is only with recent efforts to make group scheduling "just work" that it has started to come into wider use. As it happens, the kernel has a very similar feature for managing access to block I/O devices which is also, arguably, underused. In this case, though, I/O group scheduling is not as completely implemented as CPU scheduling, but some ongoing work may change that situation. 

The "completely fair queueing" (CFQ) I/O scheduler tries to divide the available bandwidth on any given device fairly between the processes which are contending for that device. "Bandwidth" is measured not in the number of bytes transferred, but the amount of time that each process gets to submit requests to the queue; in this way, the code tries to penalize ![\[Group hierarchy\]](https://static.lwn.net/images/ns/kernel/CFQ-group1.png) processes which create seek-heavy I/O patterns. (There is also a mode based solely on the number of I/O operations submitted, but your editor suspects it sees relatively little use). The CFQ scheduler also supports group scheduling, but in an incomplete way. 

Imagine the group hierarchy shown on the right; here we have three control groups (plus the default root group), and four processes running within those groups. If every process were contending fully for the available I/O bandwidth, and they all had the same I/O priority, one would expect that bandwidth to be split equally between `P0`, `Group1`, and `Group2`; thus `P0` should get twice as much I/O bandwidth as either `P1` or `P3`. If more processes were to be added to the root, they should be able to take I/O bandwidth at the expense of the processes in the other control groups. Similarly, the creation of new control groups underneath `Group1` should not affect anybody outside of that branch of the hierarchy. In current kernels, though, that is not how things work. 

With the current implementation of CFQ group scheduling, the above hierarchy is transformed into something that looks like this:
