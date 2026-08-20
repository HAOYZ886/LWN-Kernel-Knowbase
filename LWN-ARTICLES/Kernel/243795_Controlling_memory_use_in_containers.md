---
title: Controlling memory use in containers
url: https://lwn.net/Articles/243795/
date: "July 31, 2007"
category: "Containers; Memory management-Control groups"
author: "By Jonathan Corbet July 31, 2007"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 31, 2007

"Containers" is the term normally applied to a lightweight virtualization approach where all guest systems run on the host system's kernel (as opposed to running their own kernel on a special virtual machine). The container technique tends to be more efficient at run time, but it poses challenges of its own; since every container runs on the same kernel, a whole series of internal barriers must be created to give each container the illusion of having a machine to itself. The addition of these barriers to the Linux kernel has been a multi-year process as the various projects working in this area work out a set of interfaces that works for everybody. 

An important part of a complete container implementation is resource control; it is hard to maintain the fiction of a separate machine for each container if one of those containers is hogging the entire system. Extensive resource management patches have received [a chilly reception](<http://lwn.net/Articles/145135/>) in the past, but a properly done implementation based on the [process containers](<http://lwn.net/Articles/236038/>) framework might just make it in. The [CFS group scheduling](<http://lwn.net/Articles/240474/>) patch can be seen as one type of container-based resource management. But there is far more than just the CPU to worry about. 

One of the most contended resources on many systems is core memory. A container which grows without bound and forces other containers out to swap will lead to widespread grumbling on the linux-kernel list. In an effort to avoid this unfortunate occurrence, Balbir Singh and Pavel Emelianov have been working on [a container-based memory controller](<http://lwn.net/Articles/243608/>) implementation. This patch is now in its fourth iteration. 

The patch starts with a simple "resource counter" abstraction which is meant to be used with containers. It will work with any resource which can be described with simple, integer values for the maximum allowed and current usage. Methods are provided to enable hooking these counters into container objects and allowing them to be queried and set from user space. 

These counters are pressed into service to monitor the memory use by each container. Memory use can be thought of as the current resident set: the number of resident pages which processes within the container have mapped into their virtual address spaces. Unlike some previous patches, though, the current memory controller also tries to track page cache usage. So a program which is very small, but which brings a lot of data from the filesystem into the page cache, will be seen as using a lot of memory. 

To track per-container page usage, the memory controller must hook into the low-level page cache and reclaim code. It must also have a place to store information about which container each page is charged to. To that end, a new structure is created: 

```
struct meta_page {
    	struct list_head lru;
    	struct page *page;
    	struct mem_container *mem_container;
    	atomic_t ref_cnt;
        };
```

Global memory management is handled by way of two least-recently-used (LRU) lists, the hope being that the pages which have been unused for the longest are the safest ones to evict when memory gets tight. Once containers are added to the mix, though, global management is not enough. So the `meta_page` structure allows each page to be put onto a separate, container-specific LRU list. When a process within a container brings in a page and causes the container to bump up against its memory limit, the kernel must, if it is to enforce that limit, push some of the container's other pages out. When that situation arises, the container-specific LRU list is traversed to find reclaimable pages belonging to the container without having to pass over unrelated memory. 

The `page` structure in the global memory map gains a pointer to the associated `meta_page` structure. There is also a new page flag allocated for locking that structure. There is no `meta_page` structure for kernel-specific pages, but one is created for every user-space or page cache page - even for processes which have not explicitly been assigned to a container (those processes are implicitly placed in a default, global container). There is, thus, a significant cost associated with the memory controller - the addition of five pointers (one in `struct page`, four in `struct meta_page`) and one `atomic_t` for every active page in the system can only hurt. 

With this mechanism in place, the kernel is able to implement basic memory usage control for containers. One little issue remains: what happens when the kernel is unable to force a container's memory usage below its limit? In that case, the dreaded out-of-memory killer comes into play; there is a special version of the OOM killer which restricts its predations to a single container. So, in this situation, some process will die, but other containers should be unaffected. 

One interesting aspect of the problem which appears to not have been completely addressed is pages which are used by processes in more than one container. Many shared libraries will fall into this category, but much page cache usage could as well. The current code charges a page to the first container which makes use of it. Should the page be chosen to be evicted, it will be unmapped from all containers; if a different container then faults the page in, that container will be charged for it going forward. So, over time, the reclaim mechanism may well cause the charging of shared pages to be spread across the containers on the system - or to accumulate in a single, unlimited container, should one exist. Determining whether real problems could result from this mechanism will require extensive testing with a variety of workloads, and, one suspects, that effort has barely begun. 

For now we have a memory controller framework which appears to be capable of doing the core job, which is a good start. It is clearly applicable to the general container problem, but might just prove useful in other situations as well. A system administrator might not want to implement full-blown containers, but might be interested in, for example, keeping filesystem-intensive background jobs (`updatedb`, backups, etc.) within certain memory bounds. Users could put a boundary around, say, OpenOffice.org to keep it from pushing everything else of interest out of memory. There would seem to be enough value here to justify the inclusion of this patch - though a bit of work may be required first.
