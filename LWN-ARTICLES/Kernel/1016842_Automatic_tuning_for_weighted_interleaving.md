---
title: Automatic tuning for weighted interleaving
url: https://lwn.net/Articles/1016842/
date: "April 15, 2025"
category: "Memory management-Tiered-memory systems"
author: "By Jonathan Corbet April 15, 2025 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 15, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

It is common, on NUMA systems, to try to allocate all memory on the local node, since it will be the fastest. That is not the only possible policy, though; another is [weighted interleaving](<https://lwn.net/Articles/948037/>), which seeks to distribute allocations across memory controllers to maximize the bandwidth utilization on each. Configuring such policies can be challenging, though. At the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit, Joshua Hahn ran a session in the memory-management track about how that configuration might be automated. 

The core purpose of memory-allocation policy, he began, is to determine which node any given virtual memory area (VMA) should allocate pages from. One possible policy is simple round-robin interleaving to distribute allocations evenly across the system. Weighted interleaving modifies the round-robin approach to allocate more heavily from some nodes than others. Properly distributing allocations can maximize the use of the available memory bandwidth, improving performance overall. 

[![\[Joshua Hahn\]](https://static.lwn.net/images/conf/2025/lsfmm/JoshuaHahn-sm.png)](<https://lwn.net/Articles/1016843/>) The question Hahn had is: how can this interleaving be made to work out of the box? The system, he said, should provide good defaults for the interleaving weights. He had a couple of heuristics to help with the setting of those defaults. The weight ratios, intuitively, should be similar to the bandwidth ratios for the controllers involved. Bandwidth is the ultimate limit on the performance of the system, he said; it is more important than the latency of any given memory access. Weights should also be small numbers; the weight of each node is, in the end, the number of pages to allocate from that node before moving on, so smaller weights will lead to faster distribution of allocations. 

A problem arises, though, when new memory is added to the system; the kernel has to respond and recalculate all of the weights. How that should be done is not entirely clear, especially if the administrator has adjusted the defaults. The administrator should be able to tell the system what to do in that case, he said, with the available options being to recalculate all of the weights from the beginning, or to just set the weight for the new memory to one. 

Reprising a theme from [an earlier session](<https://lwn.net/Articles/1016718/>), Hahn brought up the sort of complications that hardware can bring. Given a system with two host bridges, each of which has two CXL nodes, how many NUMA nodes should the system have? The hardware can present the available resources in a few different ways, with effects that show up in the configuration problem at the kernel level. 

Ideally, of course, the tuning of the weights should be dynamic, based on some heuristic, but Hahn said that he is not entirely convinced that bandwidth use is the right metric to optimize for. He wondered if the kernel should be doing the tuning, or whether it should be delegated to user space, which might have more information. Liam Howlett said that the responsibility for this tuning definitely belongs in user space; the kernel cannot know what the user wants. Gregory Price (who did the original weighted-interleaving work) pointed out that there is currently no interface that allows one process to adjust another's weights; that would be needed for a user-space solution. 

Michal Hocko said that problems like this show that the kernel's NUMA interfaces are not addressing current needs. That problem needs to be addressed; it presents a good challenge that can lead to the creation of better interfaces. Jonathan Cameron said that user space currently does not have enough data to solve this problem. 

Price said that users may want to interleave a given VMA from the moment it is instantiated, and wondered whether the NUMA abstraction is able to handle that. Hocko answered in the negative, saying that the NUMA interface was designed for static hardware, and falls short even on current systems. The kernel's memory-policy interface is constraining; it is really time to create a new NUMA API, hopefully one that will handle CXL as well. 

Howlett said that kernel developers were never able to get the out-of-memory killer right, so now that problem is usually handled in user space. He was not convinced that the kernel community would be any more successful with interleaving policy. Hocko responded that user-space out-of-memory killers did not work well either until the [pressure-stall information interface](<https://lwn.net/Articles/759781/>) was added; before then, nobody had thought that it would be the necessary feature that would enable a solution to that problem. 

The session ran out of time; it ended with a general consensus that a better interface for controlling memory policies is needed. Now all that is needed is for somebody to actually do that work.
