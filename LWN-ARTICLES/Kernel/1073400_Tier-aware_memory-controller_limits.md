---
title: "Tier-aware memory-controller limits"
url: https://lwn.net/Articles/1073400/
date: "May 25, 2026"
category: "Memory management-Control groups; Memory management-Tiered-memory systems"
author: "By Jonathan Corbet May 25, 2026 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 25, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Joshua Hahn began his session in the memory-management track of the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) by saying that the memory controller for control groups is intended to provide resource allocation, accounting, and protection from interference by other tasks. But it was not really designed for tiered-memory systems; he is looking for a way to improve that situation. 

[![\[Joshua Hahn\]](https://static.lwn.net/images/conf/2026/lsfmm/JoshuaHahn-sm.png)](<https://lwn.net/Articles/1073406/>) Tiered-memory systems have two or more classes of memory, each of which has different performance characteristics. The system's RAM is relatively fast, but also relatively scarce, while plentiful memory on a [CXL](<https://en.wikipedia.org/wiki/Compute_Express_Link>) device may be slower. The memory controller, though, has no awareness of these differences, with the result that two control groups running the same workload under the same policies may perform differently. Without control over the physical placement of memory, the memory controller is not able to provide a consistent execution environment. 

Hahn asked, rhetorically, whether anybody cares about this problem. There are, naturally, a number of situations where people do care. Latency-sensitive workloads will suffer if they are relegated to slower memory. Hosting services want to provide fairness across all of their tenants. And, in general, performance should be predictable. It is not possible, for example, to measure performance gains from other work if the execution time of the workload is inherently variable. 

The proposed solution is to add a new memory-controller knob, `memory_tiered_limits`, that would enable tier awareness. Another set of knobs, `memory.toptier_min`, `memory.toptier_low`, `memory.toptier_high` and `memory.toptier_max`, would regulate how much memory the group is entitled to in the top memory tier, and the maximum amount that it can use. When, for example, usage reaches the `memory.toptier_high` value, reclaim on that tier would be triggered for that control group. 

This scheme, he said, yields more consistent results on tiered-memory systems for a variety of workloads. It can also improve throughput overall; distributing a workload properly across tiers can maximize the use of the available memory bandwidth for all of those tiers. 

There are some questions still to be answered, he said. Should reclaim always be triggered when top-tier usage hits the watermark, or should it only happen when there is memory pressure? He suggested that, if there is still top-level RAM available in the system, it might not make sense to limit its use. On the user-interface side, there is the question of how much tuning of tier ratios the user should be allowed to do. Promotion of folios from slower to faster tiers is a longstanding problem with the tiered-memory concept, due to the difficulty of efficiently determining _which_ folios are the right ones to promote. It is a problem here too; Hahn would like to find an efficient solution at the control-group level. 

A member of the audience asked whether it was possible to expand this mechanism to more than two tiers. He expressed concern that the `toptier` name could not be changed, leading to problems where the top tier is some sort of scarce, high-bandwidth memory rather than regular system RAM. Hahn said that he would like to see such a system before trying to adapt the memory controller to it. 

Another participant asked about fallback. Within the kernel, if an attempt to allocate top-tier memory fails, the allocator will fall back to slower memory. With this proposal, instead, the controller would trigger reclaim on the control group if the top-tier limit has been reached. Hahn agreed that this is a change in behavior, but said that is a result of introducing a new concept — tiered memory — to the memory controller. 

The session wound down with an unfocused discussion on the difficulties of tracking memory used by the networking subsystem in general.
