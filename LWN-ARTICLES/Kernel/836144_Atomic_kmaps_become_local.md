---
title: Atomic kmaps become local
url: https://lwn.net/Articles/836144/
date: "November 6, 2020"
category: "Memory management-High memory; kmap atomic"
author: "By Jonathan Corbet November 6, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 6, 2020

The `kmap()` interface in the kernel is a bit of a strange beast. It only exists to overcome the virtual addressing limitations of 32-bit CPUs, but it affects code across the kernel and has side effects on 64-bit machines as well. A recent discussion on the handling of preemption within the kernel identified a number of problems in need of attention, [one of which was the `kmap()` API](<https://lwn.net/Articles/831678/#highmem>). Now, an extension to this API called `kmap_local()` is being proposed to address some of the problems; it signals another step in the kernel community's slow move away from supporting 32-bit machines as first-class citizens. 

#### Why we have `kmap()`

A 32-bit processor will, unsurprisingly, use 32-bit pointers, which limits the amount of memory that can be addressed to 4GB. The resulting 4GB address space is split between user space and the kernel, with the kernel getting 1GB in the most common configurations; that space holds the kernel's code and data, memory-mapped I/O areas, and the "direct map" that gives the kernel access to physical memory. The direct map clearly cannot address a lot of memory; once the kernel's other needs are taken care of, there is room for significantly less than 1GB of mappings to physical memory. 

As a result, any system with 1GB or more of physical memory will have to be managed without a direct mapping to some of that memory. The memory that lies above the range that can be directly mapped is called "high memory"; on many systems, most of the installed memory is high memory. User space can use high memory without noticing any difference, but the kernel side is a bit more complicated. Whenever the kernel must access a high-memory page (to zero out a page prior to giving it to user space, for example), it must first create a temporary mapping for that page. The `kmap()` interface exists to manage these mappings. 

The `kmap()` function itself will map a given page into the kernel's address space, returning a pointer that can now be used to access the page's contents. Mappings created this way are expensive, though. They consume address space, and mapping changes must be propagated across all the CPUs of the system, which is costly. This work is necessary if a mapping must last for a relatively long time, but the bulk of high-memory mappings in the kernel are short-lived and only used in one place; the cost of `kmap()` is mostly wasted in such cases. 

Thus, the `kmap_atomic()` API was added as a way of avoiding this cost. It, too, will map a high-memory page into the kernel's address space, but with some differences. It uses one of a small set of address slots for the mapping, and that mapping is only valid on the CPU where it is created. This design implies that code holding one of these mappings must run in atomic context (thus the name `kmap_atomic()`); if it were to sleep or be moved to another CPU, confusion and data corruption would be an almost certain result. Thus, whenever code running in kernel space creates an atomic mapping, it can no longer be preempted or migrated, and it is not allowed to sleep, until all atomic mappings have been released. 

On 64-bit systems, calls to `kmap()` and `kmap_atomic()` have no real work to do; a 64-bit address space is more than sufficient to address the memory one might expect to see installed in any real-world system (for now), so all of physical memory appears in the direct map. But calling `kmap_atomic()` will disable preemption anyway, mostly as a debugging tool. It is a way of ensuring that code that sleeps while holding an atomic mapping will generate an error on 64-bit systems, meaning that such bugs are much more likely to be found before they show up on some 32-bit configuration that developers do not test. 

Disabling preemption is a red flag for realtime developers, who have worked hard for years to ensure that any given CPU can be preempted by a higher-priority task at any time. Each of the hundreds of `kmap_atomic()` call sites in the kernel creates a non-preemptable section that could be the source of unwanted latency. The last time this subject came up, there was a brief discussion of removing support for high memory from the kernel entirely; this move would simplify a lot of code and would certainly be popular, but it would also break support for existing systems that are still being shipped with new kernels. So high-memory support cannot be ripped out of the kernel quite yet. 

#### Shifting the cost

Developers are thus left in the position of having to find a second-best solution to the problem; that solution is likely to be the [`kmap_local()` patch set](<https://lwn.net/ml/linux-kernel/20201103092712.714480842@linutronix.de/>) from Thomas Gleixner. It provides a set of new functions similar to `kmap_atomic()`, but without the need to disable preemption. The new functions are: 

```
void *kmap_local_page(struct page *page);
        void *kmap_local_page_prot(struct page *page, pgprot_t prot);
        void *kmap_local_pfn(unsigned long pfn);
        void kunmap_local(void *addr);
```

The first two variants take a pointer to the `page` structure corresponding to the page of interest and return the address where the page is mapped; the second also allows the caller to specify the page protections to be applied to the mapping. If the caller has a page-frame number rather than a `page` structure, `kmap_local_pfn()` can be used. Regardless of how the mapping was created, it is destroyed with `kunmap_local()`. 

Internally, these mappings are implemented in the same way as `kmap_atomic()` — but that implementation is significantly changed by this patch set. In current kernels, each architecture has its own implementation, but almost all of the code is the same; Gleixner cleaned out this duplication and coalesced the implementations into a single, cross-architecture one. As a result, the patch set deletes over 600 lines of code while adding new functionality. 

Once the common implementation is in place, the management of the slots used for short-term mappings changes. In current kernels, they are stored in a per-CPU data structure; they are thus shared by all threads that run on the same CPU. That is one of the reasons why preemption cannot be allowed when holding an atomic mapping; a running process and the process that preempts it might both try to use the same slots, with generally displeasing results. In the new scheme, the mappings are stored in the `task_struct` structure; they are thus unique to each thread. 

The actual page-table entries that (on 32-bit systems) implement the mappings cannot be per-thread, though, so something more will have to be done to safely enable preemption in this scheme. At context-switch time, the new code looks to see whether either the outgoing or the incoming task has active local mappings; if so, those for the outgoing task are torn down and the incoming task's are reestablished. This work will slow down context switches a bit but, as Gleixner noted: ""That's obviously slow, but highmem is slow anyway"". 

Local page mappings are still only established on the local CPU, meaning that a process holding such mappings cannot be migrated without asking for trouble. Thus, while preemption remains enabled when kernel code creates a local mapping, migration from one CPU to another is disabled. It's worth noting that current kernels don't have the machinery to disable migration in this way; that is a feature that has been limited to the realtime kernels so far. Peter Zijlstra has been working on [a migration-disable implementation](<https://lwn.net/ml/linux-kernel/20201023101158.088940906@infradead.org/>) for the general case that has not yet been merged; it is obviously a prerequisite for the `kmap_local()` work. 

Once everything is in place, the only difference between `kmap_atomic()` and `kmap_local()` will be the execution context when holding a mapping. Atomic mappings still disable preemption, while local mappings only disable migration. Otherwise, the two mapping types are identical. That leads to an obvious question: why not just switch everybody to `kmap_local()`? That is indeed the long-term plan, but there is a little hitch: some `kmap_atomic()` callers almost certainly depend on preemption being disabled, perhaps without the developer even being aware of it. So every one of hundreds of call sites will need to be audited and converted, one by one. 

That work can be expected to take a while, but there should eventually be a time when `kmap_atomic()` is no longer used and can be removed from the kernel. The newer API preserves functionality for 32-bit systems, but it shifts some of the cost toward those systems and away from the 64-bit systems that dominate the computing landscape now. It's not the removal of high-memory support, but it is a sign that systems using high memory are increasingly seen as a niche use case that will not be supported forever.
