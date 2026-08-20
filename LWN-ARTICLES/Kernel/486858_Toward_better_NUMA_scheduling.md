---
title: Toward better NUMA scheduling
url: https://lwn.net/Articles/486858/
date: "March 16, 2012"
category: "Memory management-NUMA systems; NUMA; Scheduler-NUMA"
author: "By Jonathan Corbet March 16, 2012"
---

By **Jonathan Corbet**  
March 16, 2012

A non-uniform memory access (NUMA) system is a computer divided into "nodes," where each node (which may contain multiple processors) has some memory which is local to the node. All system memory is visible to all nodes, but accesses to memory that is not local to the accessing node must go over an inter-node bus; as a result, non-local accesses are significantly slower. There is, thus, a real performance advantage to be gained by keeping processes and their memory on the same node. 

The Linux kernel has had NUMA awareness for some time, in that it understands that moving a process from one node to another can be an expensive undertaking. There is also an interface (available via the `mbind()` system call) by which a process can request a specific allocation policy for its memory. Possibilities include requiring that all allocations happen within a specific set of nodes (`MPOL_BIND`), setting a looser "preferred" node (`MPOL_PREFERRED`), or asking that allocations be distributed across the system (`MPOL_INTERLEAVE`). It is also possible to use `mbind()` to request the active migration of pages from one node to another. 

So NUMA is not a new concept for the kernel, but, as Peter Zijlstra noted in [the introduction to a large NUMA patch set](<https://lwn.net/Articles/486850/>), things do not work as well as they could: 

Current upstream task memory allocation prefers to use the node the task is currently running on (unless explicitly told otherwise, see mbind()/set_mempolicy()), and with the scheduler free to move the task about at will, the task's memory can end up being spread all over the machine's nodes. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

While the scheduler does a reasonable job of keeping short running tasks on a single node (by means of simply not doing the cross-node migration very often), it completely blows for long-running processes with a large memory footprint. 

As might be expected, the patch set is dedicated to the creation of a kernel that does not "completely blow." To that end, it adds a number of significant changes to how memory management and scheduling are done in the kernel. 

There are three major sub-parts to Peter's patch set. The first is a reworked patch set first [posted](<http://markmail.org/message/mdwbcitql5ka4uws>) by Lee Schermerhorn in 2010. These patches change the memory policy mechanism to make it easier for the kernel to fix things up after a process's memory has been allocated on distant nodes. "Page migration" is the process of moving a page from one node to another without the owning process(es) noticing the change. With Lee's patches, the kernel implements a variation called "lazy migration" that does not immediately relocate any pages. Instead, the target pages are simply unmapped from the process's page tables, meaning that the next access to any of them will generate a page fault. Actual migration is then done at page fault time. Lazy migration is a less expensive way of moving a large set of pages; only the pages that are actually used are moved, the work can be spread over time, and it will be done in the context of the faulting process. 

The lazy migration mechanism is necessary for the rest of the patch set, but it has value on its own. So the feature is made available to user space with the `MPOL_MF_LAZY` flag; it is intended to be used with the `MPOL_MF_MOVE` flag, which would otherwise force the immediate migration of the affected pages. There is also a new `MPOL_MF_NOOP` flag allowing the calling process to request the migration of pages according to the current policy without changing (or even knowing) that policy. 

With lazy migration, memory distributed across a system as the result of memory allocation and scheduling decisions can be slowly pulled back to the optimal node. But it is better to avoid making that kind of mess in the first place. So the second part of the patch set starts by adding the concept of a "home node" to a process. Each process (or "NUMA entity" - meaning groups containing a set of processes) is assigned a home node at `fork()` time. The scheduler will then try hard to avoid moving a process off its home node, but within bounds: a process will still be run on a non-home node if the alternative would be an unbalanced system. Memory allocations will, by default, be performed on the home node, even if the process is running elsewhere at the time. 

These policies should minimize the scattering of memory across the system, but, with this kind of scheduling regime, it is inevitable that, eventually, one node will end up with too many processes and too little memory while others are underutilized. So, sometimes, it will be necessary to rebalance things. When the scheduler notices that long-running tasks are being forced away from their home nodes - or that they are having to allocate memory non-locally - it will consider migrating them to a new node. Migration is not a half-measure in this case; the scheduler will move both the process and its memory (using the lazy migration mechanism) to the target node. The move is expensive, but the process (and the system) should run much more efficiently once it's done. It only makes sense for processes that are going to be around for a while, though; the patch set tries to approximate that goal by only considering processes with at least one second of run time for migration. 

The final piece is a pair of new system calls allowing processes to be put into "NUMA groups" that will share the same home node. If one of them is migrated, the entire group will be migrated. The first system call is: 

```
int numa_tbind(int tid, int ng_id, unsigned long flags);
```

This system call will bind the thread identified by `tid` to the NUMA group identified by `ng_id`; the `flags` argument is currently unused and must be zero. If `ng_id` is passed as `MS_ID_GET`, the system call will, instead, simply return the current NUMA group ID for the given thread. A value of `MS_ID_NEW`, instead, creates a new NUMA group, binds the thread to that group, and returns the new ID. 

The second new system call is: 

```
int numa_mbind(void *addr, unsigned long len, int ng_id, unsigned long flags);
```

This call will set up a memory policy for the region of `len` bytes starting at `addr` and bind it to the NUMA group identified by `ng_id`. If necessary, lazy migration will be used to move the memory over to the node where the given NUMA group is based. Once again, `flags` is unused and must be zero. Once the memory is bound to the NUMA group, it will stay with the processes in that group; if the processes are moved, the memory will move with them. 

Peter provided some benchmark results from a two-node system. Without the NUMA balancing patches, over time, the benchmark ended up with just as many remote memory accesses as local accesses - allocated memory was spread across the system. With the NUMA balancer, 86% of the memory accesses were local, leading to a significant speedup. As Peter put it: ""These numbers also show that while there's a marked improvement, there's still some gain to be had. The current numa balancer is still somewhat fickle."" A certain amount of fickleness is perhaps to be expected for such an involved patch set, given how young it is. Given some time, reviews, and testing, it should evolve into a solid scheduler component, giving Linux far better NUMA performance than it has ever had in the past.
