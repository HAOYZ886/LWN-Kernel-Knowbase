---
title: "Top-tier memory management"
url: https://lwn.net/Articles/857133/
date: "May 28, 2021"
category: "Memory management-Tiered-memory systems"
author: "May 28, 2021 This article was contributed by Marta Rybczyńska"
---

May 28, 2021

This article was contributed by Marta Rybczyńska

Modern computing systems can feature multiple types of memory that differ in their performance characteristics. The most common example is NUMA architectures, where memory attached to the local node is faster to access than memory on other nodes. Recently, persistent memory has started appearing in deployed systems as well; this type of memory is byte-addressable like DRAM, but it is available in larger sizes and is slower to access, especially for writes. This new memory type makes memory allocation even more complicated for the kernel, driving the need for a method to better manage multiple types of memory in one system.

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

NUMA architectures contain some memory that is close to the current CPU, and some that is further away; remote memory is typically attached to different NUMA nodes. There is a difference in access performance between local and remote memory, so the kernel has gained support for NUMA topologies over the years. To maximize NUMA performance, the kernel tries to keep pages close to the CPU where they are used, but also allows the distribution of virtual memory areas across the NUMA nodes for deterministic global performance. The [kernel documentation](<https://www.kernel.org/doc/html/latest/admin-guide/mm/numa_memory_policy.html>) describes ways that tasks may influence memory placement on NUMA systems.

The NUMA mechanism [can be extended](<https://lwn.net/Articles/787418/>) to handle persistent memory as well, but it was not really designed for that case. The future may bring even more types of memory, such as [High Bandwidth Memory (HBM)](<https://en.wikipedia.org/wiki/High_Bandwidth_Memory>), which stacks DRAM silicon dies and provides a larger memory bus. Sooner or later, it seems that a different approach will be needed. 

Recently, kernel developers have been discussing a possible solution to the problem of different memory types: adding the notion of memory tiers. The proposed code extends the NUMA mode to include features like migrating infrequently used pages to slow memory, migrating hot pages back to fast memory, and a proposal for a control mechanism for this feature. The changes to the memory-management subsystem to support different tiers are complex; the developers are discussing three related patch sets, each building on those that came before.

#### Migrating from fast to slow memory

The first piece of the puzzle takes the form of [a patch set](<https://lwn.net/ml/linux-kernel/20210401183216.443C4443%40viggo.jf.intel.com/>) posted by Dave Hansen. It improves the memory reclaim process, which normally kicks in when memory is tight and pushes out the content of rarely used pages. Hansen said that, in a system with persistent memory, those pages could instead be migrated from DRAM to the slower memory, maintaining access to them if they are needed again. Hansen noted in the cover letter that this is a partial solution, as migrated pages will be stuck in slow memory with no path back to faster DRAM. This mechanism is optional and users will be able to [enable it](<https://lwn.net/ml/linux-kernel/20210401183235.BCC49E8B@viggo.jf.intel.com/>) on-demand with the sysctl `vm.zone_reclaim_mode` or in `/proc/sys/vm/zone_reclaim_mode` with the bitmask set to 8.

The patch set received some initial positive reviews, including one [from Michal Hocko](<https://lwn.net/ml/linux-kernel/YHmEm/yHpaqO6khp@dhcp22.suse.cz/>), who noted that the feature could also be useful in traditional NUMA systems without memory tiers.

#### ...and back

The second part of the puzzle is a migration of frequently used pages from slow to fast memory. This has been [proposed](<https://lwn.net/ml/linux-kernel/20210311081821.138467-2-ying.huang%40intel.com/>) in a patch set by Huang Ying.

In current kernels, NUMA balancing works by periodically unmapping a process's pages. When there is a page fault caused by access to an unmapped page, the migration code decides whether the affected page should be moved to the memory node where the page fault occurred. The migration decision depends on a number of factors, including frequency of access and the availability of memory on the accessing node. 

The [proposed patch](<https://lwn.net/ml/linux-kernel/20210311081821.138467-5-ying.huang@intel.com/>) takes advantage of those page faults to make a better estimation of which pages are hot; it replaces the current algorithm, which considers the most recently accessed pages to be hot. The new estimate uses the elapsed time between the time the page was unmapped and the page fault as a measure of how hot the page is, and offers a sysctl knob to define a threshold: `kernel.numa_balancing_hot_threshold_ms`. All pages with a fault latency lower than the threshold will be considered hot. Correctly setting this threshold may be difficult for the system administrator, so the [final patch](<https://lwn.net/ml/linux-kernel/20210311081821.138467-7-ying.huang@intel.com/>) of the series implements a method to automatically adjust it. To do that, the kernel monitors the number of pages being migrated with the user-configurable balancing rate limit `numa_balancing_rate_limit_mbps`, then it increases or decreases the threshold to bring the rate closer to that value. 

#### Controlling memory tiers

Finally, Tim Chen [submitted](<https://lwn.net/ml/linux-kernel/cover.1617642417.git.tim.c.chen@linux.intel.com/>) a proposal for management of the configuration of memory tiers, and the top-level tier containing the fastest memory in particular. The proposal is based on control groups version 1 (Chen noted that support of version 2 is in the works), and monitors the amount of top-tier memory used by the system and by each control group individually. It uses soft limits and enables kswapd to move pages to slower memory in control groups that exceed their soft limit. Since the limit is soft, a control group may be allowed to exceed the limit if top-tier memory is plentiful, but may be quickly cut back to the limit if that resource is tight. 

In Chen's proposal, the soft limit for a control group is called `toptier_soft_limit_in_bytes`. The code also traces the global usage of top-tier memory, and if the free memory falls under `toptier_scale_factor`/10000 of the overall memory of the node it is attached to, kswapd will start memory reclaim focused on control groups that exceed their soft limit.

Hocko [disliked the idea of using soft limits](<https://lwn.net/ml/linux-kernel/YGwlGrHtDJPQF7UG@dhcp22.suse.cz/>): 

> In the past we have learned that the existing implementation is unfixable and changing the existing semantic impossible due to backward compatibility. So I would really prefer the soft limit just find its rest rather than see new potential use cases. 

The likely reasons for Hocko's dislike for soft limits come from the previous attempts to change the interface (LWN [covered](<https://lwn.net/Articles/592045/>) the discussions in 2013 and 2014). The default soft limit is "unlimited", and this cannot be changed without a risk of backward compatibility issues. 

Further into the discussion, Shakeel Butt [asked](<https://lwn.net/ml/linux-kernel/CALvZod7StYJCPnWRNLnYQV8S5CBLtE0w4r2rH-wZzNs9jGJSRg@mail.gmail.com/>) about a use case where high-priority tasks would obtain better access to the top-tier memory, which would be more strictly limited for low-priority tasks. Yang Shi [pointed](<https://lwn.net/ml/linux-kernel/CAHbLzkrPD6s9vRy89cgQ36e+1cs6JbLqV84se7nnvP9MByizXA@mail.gmail.com/>) to [earlier work](<https://lwn.net/Articles/787418/>) that divided fast and slow memory for different tasks, and concluded that the solution was hard to use in practice, as it required good knowledge of the hot and cold memory in the specific system. The developers discussed more fine-grained control of the type of memory used, but did not reach a conclusion. 

Before the discussion stopped, Hocko [offered some ideas](<https://lwn.net/ml/linux-kernel/YG7ugXZZ9BcXyGGk@dhcp22.suse.cz/>) on how the interface could work: differing types of memory would be configured into separate NUMA nodes, and tasks could indicate their preferences for which nodes should host their memory. Some nodes might be reclaimed ahead of others when memory pressure hits. He also further [noted](<https://lwn.net/ml/linux-kernel/YHkw8Ou2VAgHYTjl@dhcp22.suse.cz/>) that this mechanism should be generic, not based on the location of persistent memory in the CPU nodes: 

> I haven't been proposing per NUMA limits. [...] All I am saying is that we do not want to have an interface that is tightly bound to any specific HW setup (fast RAM as a top tier and PMEM as a fallback) that you have proposed here. We want to have a generic NUMA based abstraction. 

#### Next steps

None of the patch sets have been merged at the moment of this writing, and it looks like it is not going to happen soon. Changes in memory management take time and it seems that the developers need to agree on the way to control the usage of fast/slow memory in different workloads before a solution will appear. The top-tier management patches are explicitly intended as discussion fodder and are not intended for merging in their current form in any case. We will likely see more discussion on the subject in the coming months.
