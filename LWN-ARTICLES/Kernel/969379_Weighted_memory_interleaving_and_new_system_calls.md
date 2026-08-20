---
title: Weighted memory interleaving and new system calls
url: https://lwn.net/Articles/969379/
date: "April 19, 2024"
category: "Memory management-Tiered-memory systems"
author: "By Daroc Alden April 19, 2024"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Daroc Alden**  
April 19, 2024

Gregory Price recently posted version 4 of [ a patch set](<https://lwn.net/ml/linux-kernel/20231223181101.1954-1-gregory.price@memverge.com/>) that adds support for weighted memory interleaving — allowing a process's memory to be distributed between [ non-uniform memory access](<https://en.wikipedia.org/wiki/Non-uniform_memory_access>) (NUMA) nodes in a more controlled way. According to the performance measurements he includes, the patch set could provide a significant improvement for computers with network-attached memory. The patch set also introduces new system calls and paves the way for future extensions intended to give processes more control over their own memory. 

Modern computers can have a variety of kinds of memory in use at the same time. Not just traditional NUMA between separate banks of RAM within the same computer, but also memory distributed across a data center, like [ Compute Express Link](<https://en.wikipedia.org/wiki/Compute_Express_Link>) (CXL) attached memory. These technologies allow computers to support much larger amounts of memory, at the cost of significantly complicating memory management and slower memory access speeds. 

Current Linux kernels group different kinds of memory into tiers based on their latency. LWN [covered](<https://lwn.net/Articles/948037/>) how they interact with an earlier version of Price's patch set in October. The kernel also allows configuring processes to have different pages of their memory resident on different NUMA nodes. This spreads out the load between the separate parts of memory, but it's not perfect. For one thing, banks of memory can have different available bandwidths. The current default behavior is to assign allocated pages to different nodes in a round-robin way, which could over-allocate the bandwidth of the least-capable bank, even if other banks have more available capacity. 

Price's patch set lets users specify unique weights for each NUMA node, and uses those weights when distributing freshly allocated pages. These weights are configured globally, but can be applied to specific processes using the kernel's NUMA [ memory-policy](<https://www.kernel.org/doc/html/latest/admin-guide/mm/numa_memory_policy.html>) support. Only tasks that have the new `MPOL_WEIGHTED_INTERLEAVE` memory policy will use the weights. 

The cover letter of the patch set includes a performance comparison (contributed by several different people) demonstrating how much better weighted interleaving can perform than the default round-robin scheme. In brief, it compares four settings for the same workloads: plain DRAM, CXL attached memory with the default interleaving policy, CXL memory with global weights according to bandwidth, and "targeted" weights that use different settings for the executable code, stack, and heap of the process. The default interleaving policy is on average 78% slower than DRAM. The global weights bring that performance to between 6% slower and 4% faster than DRAM depending on workload, and correctly chosen targeted weights push the performance to 2.5% to 4% better than DRAM. 

Targeted weights have such a dramatic effect because different areas of a process's memory can have different access patterns that give an advantage to one memory policy or another. Memory policies for a whole process or a specific area of memory are configured with [ `set_mempolicy()`](<https://www.man7.org/linux/man-pages/man2/set_mempolicy.2.html>) and [ `mbind()`](<https://man7.org/linux/man-pages/man2/mbind.2.html>) respectively: 

```
long set_mempolicy(int mode, const unsigned long *nodemask,
                           unsigned long maxnode);
    
        long mbind(void addr[.len], unsigned long len, int mode,
                   const unsigned long nodemask[(.maxnode + ULONG_WIDTH - 1)
                                                / ULONG_WIDTH],
                   unsigned long maxnode, unsigned int flags);
```

The signature of `mbind()` introduces problems for weighted memory interleaving, however; the signature cannot be extended, because it is running up against the limits of how many arguments can be provided to a system call (at most six). Price's patch set rectifies this by [ introducing a new system call](<https://lwn.net/ml/linux-kernel/20231223181101.1954-11-gregory.price@memverge.com/>) — `mbind2()` — that takes a structure as an argument, but otherwise performs the same function. 

```
struct mpol_args {
          __u16 mode;
          __u16 mode_flags;
          __s32 home_node;
          __u64 pol_maxnodes;
          __aligned_u64 *pol_nodes;
          /* Optional: interleave weights for MPOL_WEIGHTED_INTERLEAVE */
          unsigned char *il_weights;    /* of size MAX_NUMNODES */
        };
    
        mbind2(unsigned long addr, unsigned long len, struct mpol_args *args,
               size_t size, unsigned long flags);
```

The `mpol_args` struct is intended to be extensible over time, which is why the system call includes the structure's size. Any new extensions to the memory-policy infrastructure can add options to the end of the structure, and old callers won't need to be updated. Price's patch set also adds `set_mempolicy2()` and `get_mempolicy2()` using the same scheme, to support setting or retrieving task-wide weights, respectively. 

Like existing memory-policy settings, Price's new weighted-interleaving policy is not a hard-and-fast rule. Setting a new memory policy does not migrate existing pages (unless the `MPOL_MF_MOVE` option is specified), and if the preferred NUMA node has no more memory available, the kernel will fall back to using the next available NUMA node. Price [ spelled this out explicitly](<https://lwn.net/ml/linux-kernel/ZYqEjsaqseI68EyJ@memverge.com/>) in response to concerns brought up during review: 

> This interface does not limit memory usage of a particular node, it distributes data according to the requested policy. 
> 
> Nuanced distinction, but important. If nodes become exhausted, tasks are still free to allocate memory from any node in the nodemask, even if it violates the requested mempolicy. 
> 
> This is consistent with the existing behavior of mempolicy. 

The weighted-interleaving patch set has gathered relatively little commentary, perhaps because the idea itself has been in progress for a long while. Price [ posted a related patch set](<https://lwn.net/ml/linux-kernel/20231122211200.31620-1-gregory.price@memverge.com/>) in November that would have changed `set_mempolicy()` to take a process ID as an additional argument. The change would allow privileged processes to set memory policies for other processes. At the time, Price described the November patch set as being designed ""to make mempolicy more flexible and extensible, such as adding interleave weights (which may need to change at runtime due to hotplug events)"". Because `mbind()` already passes six parameters, however, that change would have needed new system calls as well. The November patch set did not end up being merged. Now that Price's newer weighted-interleaving patch set introduces the needed system calls, it is possible that another version of the older patch set will follow once the weighted-interleaving one is accepted. 

Price's weighted-interleaving patch set does seem likely to be merged [Update: a reader [points out](<https://lwn.net/Articles/970546/>) that some of these changes, but not the new system calls, were merged under a different name as part of 6.9], given the impressive number of Suggested-by tags in the cover letter and the minimal objections from Ying Huang and Geert Uytterhoeven, who reviewed it. It seems as though many people are eager to have more control over how their processes' memory is distributed.
