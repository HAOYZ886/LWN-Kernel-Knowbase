---
title: Proactive reclaim for tiered memory and more
url: https://lwn.net/Articles/894849/
date: "May 13, 2022"
category: "Memory management-Tiered-memory systems"
author: "By Jonathan Corbet May 13, 2022 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 13, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

Memory reclaim in Linux is largely a reactive practice; the kernel tries to find memory it can repurpose in response to the amount of free memory falling too low. Developers have often wondered if a proactive reclaim mechanism might lead to better performance, for some workloads at least, and optimal use of tiered-memory systems will likely require more active reclamation of memory as well. At the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Davidlohr Bueso led a brief session on the topic. 

Bueso started by suggesting the addition of a per-node knob that would enable proactive reclaim; an administrator would write a number indicating the amount of memory that should be reclaimed, and the kernel would attempt to make it happen. It might also be possible, he said, to extend the debugfs knob added by the [multi-generational LRU patches](<https://lwn.net/Articles/856931/>) rather than adding a new knob. Michal Hocko opposed that latter idea, though, saying that he did not want to make anything in debugfs into an API that would have to be maintained. [![\[Davidlohr Bueso\]](https://static.lwn.net/images/conf/2022/lsfmm/DavidlohrBueso-sm.png)](<https://lwn.net/Articles/894857/>)

Instead, Hocko said, a knob of this sort should be put into sysfs. There are two ideas for how this control could work: there could be a single knob that would accept a mask indicating which nodes to target for reclaim, or there could be a per-node knob as described by Bueso. Hocko likes the per-node knob idea better, since it provides better control to the administrator. Johannes Weiner said that he has tried to add a similar sort of knob to the control-group memory controller; it would accept a count of pages to reclaim from a given group. That controller does round-robin reclaim across the processes contained within the group, which might be good enough, he said. He suggested testing this mechanism on tiered-memory systems to see how it works. 

Bueso asked whether that sort of interface can be counted on to work in the future; not every system uses control groups in this way, and control at the global level might be handled differently. Weiner said that users want all of the features in both the global and control-group settings, so there should not be any divergence there. 

Another attendee pointed out a couple of other use cases for proactive reclaim. Migration of virtual machines will go faster if there are fewer pages to copy, so administrators would like to be able to force a virtual machine to reclaim as much memory as possible before the migration begins. The virtual machine can report which pages have been freed to the hypervisor, and those pages can be left out of the copy to the new host. A similar use case is suspend-to-disk, which will happen more quickly if there are a lot of free pages that need not be written to persistent storage. 

Bueso turned the topic to testing of proactive-reclaim mechanisms; there are a lot of ideas going around, he said, but not a lot of numbers showing how well they actually work. For example, he likes the hot-page selection algorithm that is part of the [tiered-memory work](<https://lwn.net/Articles/893024/>), but there is only one benchmark result that gives any information on its performance. The minimal approach to benchmarking appears to be the standard for this kind of work, he said, and that worries him. 

He continued with a request for an easier way to subject a patch set to a variety of workloads. He has been hacking on [MMTests](<https://github.com/gormanm/mmtests>) toward that end, trying to get an indication of just when a workload starts to push pages out of DRAM and into a slower memory tier. That helps to know whether the tiering algorithm is actually working, he said. But he would like to find a way to add tests that exercise the memory-management subsystem in ways beyond just consuming lots of RAM. 

As the session wound down, he also said that he would like a way to export the kernel's view of the various memory tiers to user space. The consensus seemed to be that a sysfs file should be added for that purpose.
