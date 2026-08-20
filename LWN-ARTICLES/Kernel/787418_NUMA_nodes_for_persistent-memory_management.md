---
title: "NUMA nodes for persistent-memory management"
url: https://lwn.net/Articles/787418/
date: "May 6, 2019"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet May 6, 2019 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 6, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

While persistent memory is normally valued for its persistence, there is also a subcurrent of interest in using it in settings where persistence is not important. In particular, the fact that this memory is relatively inexpensive makes it appealing to use instead of ordinary RAM in budget-conscious settings. At the 2019 Linux Storage, Filesystem, and Memory-Management Summit, two sessions in the memory-management track looked at how the kernel's NUMA mechanism could be pressed into service to manage non-persistent uses of persistent memory. 

#### Persistent memory as RAM

Yang Shi led the first session, which was focused on a scheme to use persistent memory configured into CPUless NUMA nodes as RAM for low-cost virtual machines. The idea seems to have some merit, but there are a number of details to be worked out yet. 

The motivation behind this work is simple enough: offer cheaper virtual machines for customers who are willing to put up with slower memory. It is an idea that has been [making the rounds](<https://lwn.net/Articles/777212/>) for [![\[Yang Shi\]](https://static.lwn.net/images/conf/2019/lsfmm/YangShi-sm.jpg)](<https://lwn.net/Articles/787431/>) some months already. Ideally, it would be possible to give each virtual machine a certain amount of RAM for its hottest data, while filling in the rest with persistent memory, maintaining a set ratio of the amounts of each type of memory. Virtual machines would preferentially use their available RAM up to their quota, after which data would spill over into persistent memory; the kernel would migrate data back and forth in an attempt to keep the most frequently used memory in RAM. 

Needless to say, there are some challenges involved in implementing a scheme like this. Workloads can have random and frequently changing access patterns that are hard to keep up with. Maintaining the right ratio between the two types of memory will require continuous scanning of the virtual machine's address space. It's even hard to define a service-level agreement so that customers know what sort of performance to expect. 

Michal Hocko asked what sort of API was envisioned to control this memory-type ratio. There is no control planned for customers; administrators would have a `/proc` file where the ratio could be set. Customers would be allowed some influence over where data is placed by using the `madvise()` system call to indicate which pages should be in fast memory. Dave Hansen asked about that aspect of things; given that the memory types will be split into different NUMA nodes, the existing NUMA mechanisms could be used to control placement. Mel Gorman agreed, saying that, if the kernel's NUMA awareness could be augmented to allow the specification of a percentage-based allocation strategy, the needed flexibility would be there. 

Hocko then wondered about explicit placement of pages, something that user space can request now. If the kernel then tries to shuffle things around, the situation could get messy. He would like to avoid reproducing the sort of problems we see now with CPU sets conflicting with explicit NUMA placement, which he described as a "giant mess". 

Gorman suggested that perhaps the memory controller could be augmented with per-node counters, though it would be a non-trivial problem to get right. It's not clear how memory reclaim could be handled. But it is important to realize, he said, that there are two different problems to be solved here: implementing the memory ratio and migrating pages between memory types. The first step is to get the accounting right so that the ratio can be implemented; after that, it will make sense to worry about locality. Proper accounting will require tracking memory usage on a per-node, per-process (or, more likely, per-control-group) basis, which the kernel doesn't do now. When this is implemented, he said, it's important to not make it specific to persistent memory; this mechanism could be used to express policies for different classes of memory in general. 

The discussion wandered around the accounting issue for some time without making any real progress, but it was agreed that getting the accounting working was an important first step. Control groups already have some of the needed support; finishing the job should be feasible. The data-migration task might prove harder, though; Hocko said that it would probably have to be implemented in user space. Gorman added that, once the accounting is available, the kernel provides everything that is needed to implement a brute-force migration solution in user space, though he described it with terms like "brutal" and "horrible". 

That still leaves the problem of kernel memory, though, which is not accounted in the same way and which cannot generally be migrated at all, much less from user space. Where, Hansen asked, would the dentry cache live on such a system? Putting it into slow memory would be painful. Hansen argued for supporting placement in a general way, for now. He also noted that the memory situation is becoming more complicated; slow memory may have fast caches built into it, for example. Christoph Lameter mentioned the possibility of high-end memory of the type currently found on GPUs coming to CPUs in the near future. The kernel will need as much flexibility as possible to be able to handle the more complex memory architectures on the horizon. 

The discussion returned to the ratio-enforcement problem, with Gorman repeating that control groups offer some of the needed statistics now. Hocko agreed, saying that this support is not complete, but it's a place to start. The charging infrastructure can be made more complete, and kernel-memory accounting could be added. If the result turns out not to be usable for this task, it least it would be a starting point from which developers could figure out what the real solution should look like. Gorman cautioned against premature optimism, noting the previous ideas along these lines have not succeeded. 

This kind of policy would bring some new challenges to the accounting code, Hocko said. The current memory controller works by first allocating memory in response to a request, then attempting to charge the memory to the appropriate control group. If the charge fails (the group is above its limit), the newly allocated memory is freed rather than handed over and the allocation request fails. In a ratio-based scheme, a charging failure would have to lead to an allocation being retried with a different policy instead. 

Gorman said that might not necessarily be the case; instead, if a group has moved away from its configured ratio, memory could be reclaimed from that group to bring things back into balance. There would be problems with such a solution, but it would be a place to start. Dan Williams, instead, suggested a scheme where allocations come from a random node, with the choice being biased toward the desired ratio. The session ran out of time at this point, ending with no conclusions but with a number of ideas for developers to try. 

#### Persistent memory in NUMA nodes

At this point, leadership of the discussion shifted over to Hocko and the topic moved to the use of the NUMA infrastructure for the management of persistent memory in general. There are schemes floating around to do things like configure persistent memory as a "distant" NUMA node and implement proactive reclaim algorithms that would automatically move cold pages over to that "node", which would be hidden from the memory-management subsystem otherwise. Many of these proposals, Hocko said, make the mistake of treating persistent memory as something special — an approach that he has been pushing back on. They risk creating a solution that works for today's technology but not tomorrow's. 

Instead, he would really rather just think in terms of NUMA workloads; the system is already built to handle memory with different performance characteristics, after all. Perhaps all that is really needed is to create a [![\[Michal Hocko\]](https://static.lwn.net/images/conf/2019/lsfmm/MichalHocko-sm.jpg)](<https://lwn.net/Articles/787432/>) new memory-reclaim mode that migrates memory rather than reclaiming its pages; the NUMA balancing code could then handle the task of ensuring that pages are in the right places. 

Hansen wondered how the ordering of nodes would be handled. Currently, NUMA balancing works in terms of distance — how long it takes to access memory on any given node. But that might not be optimal for the sorts of things people want to do with persistent memory. Perhaps the kernel needs to create a fallback list of places a page could be put, using the node distance as the metric initially. Gorman, though, said that was too complicated; memory placement should fall back to exactly one node at the outset. That would avoid traps like page-migration cycles, where pages would shift around the system but never find a permanent home. 

Hocko asked if the migration-based reclaim mode seemed like a good idea; it could be implemented without big changes in a simple mode to see how well it works. All that would be needed, he said, would be a hook into one place in the reclaim code. Gorman thought it made sense, but he repeated that there should only be one node to which pages could be migrated. If all nodes in the system were on a fallback list, a page would have to move through every possible option — each RAM-based node and each persistent-memory node — before actually being reclaimed. It would be necessary to maintain the history of where each page has been, and would be likely to disrupt other workloads on the system. 

Hansen thought that perhaps the problem could be addressed by creating directional paths between nodes, so that pages would only migrate in one direction. Hocko said that there was no real need for that, since the kernel would not move pages from a CPUless persistent-memory node to a RAM node, but Gorman argued that this was an unfortunate application of special-casing. One should not, he said, assume that persistent-memory nodes will never have CPUs in them. 

Once migration is solved, there is still the problem of moving pages in the other direction: how will frequently used pages be promoted to fast memory? Gorman said that the NUMA-balancing code does that now, so things should just work, but Rik van Riel argued that NUMA balancing would not work in this case; when pages are stored on a CPUless node, _all_ accesses will be remote, so NUMA balancing will naturally pull every page off of that node. Gorman conceded the point, but said that this behavior might be reasonable; pages that are being used should move to faster memory, but Van Riel said that there is no way to tell between occasional uses and frequent uses. 

Hocko was less worried about this scenario. A page that has found its way to a persistent-memory node has been demoted there by the memory-management code, meaning that it fell off the far end of the least-recently-used list. That means it hasn't been used for a while and is unlikely to be accessed again unless the access pattern changes. But Van Riel was still worried that the system could thrash through the demoted memory, and that some sort of rate limiting might be needed. Gorman said that there may be a need to alter the two-reference rule that is currently used to move pages between NUMA nodes, but that is not a radical change; the idea of using NUMA balancing is still sound, he said. 

The session wound down with a general agreement that the migrate-on-reclaim mode was an idea worth pursuing, and that relying on NUMA balancing to promote pages back to RAM should be tried. Once the simplest solution has been implemented it can be observed with real workloads; developers can go back and improve things if this approach behaves poorly.
