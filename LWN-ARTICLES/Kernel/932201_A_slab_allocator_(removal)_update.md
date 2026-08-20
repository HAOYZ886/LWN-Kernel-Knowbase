---
title: A slab allocator (removal) update
url: https://lwn.net/Articles/932201/
date: "May 22, 2023"
category: "Memory management-Slab allocators"
author: "By Jonathan Corbet May 22, 2023 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 22, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The kernel developers try hard to avoid duplicating functionality in the kernel, which is enough of a challenge to maintain as it is. So it has often seemed out of character for the kernel to support three different slab allocators (called SLAB, SLOB, and SLUB), all of which handle the management of small memory allocations in similar ways. At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), slab maintainer Vlastimil Babka updated the group on progress toward the goal of reducing the number of slab allocators in the kernel and gave an overview of what to expect in that area. 

Babka started by saying that [his original proposal](<https://lwn.net/ml/linux-mm/4b9fc9c6-b48c-198f-5f80-811a44737e5f@suse.cz/>) for the session mentioned the SLOB allocator in the title. This allocator, which was optimized for memory-limited systems, [has been on the chopping block](<https://lwn.net/Articles/918344/>) for a while now. That removal, he announced to applause, happened during the 6.4 merge window. There is a set of configuration options that can be selected to make the SLUB allocator more suitable for small-memory systems. It is now possible to call `kfree()` on all slab-allocated objects — something that SLOB never supported. 

The next step, he said, might be to remove SLAB. That would solve one of his biggest problems: he never figured out how to pronounce SLAB and SLUB so that others could hear the difference. SLAB contains 4,000 lines of code, he said, not all of which is regularly or well tested. He has found parts of the SLAB allocator that have been broken for years. Keeping SLAB around means maintaining a common-code layer used also by SLUB, which complicates maintenance. It also requires reimplementing features; both allocators have implementations of memory control groups, for example, while realtime preemption is only supported by SLUB. 

[![\[Vlastimil Babka\]](https://static.lwn.net/images/conf/2023/lsfmm/VlastimilBabka-sm.png)](<https://lwn.net/Articles/932206/>) This is not the first time somebody has suggested removing SLAB; he found at least three other times that the idea has come up. Each time the idea is raised, somebody complains about performance regressions when using SLUB. He wanted to know if the same objections would be raised this time. 

David Rientjes is one of the developers who has objected to the removal of SLAB in the past. Speaking from a Google perspective, he said that things have come a long way since then. Per-cpu partial slabs help a lot. He has been looking at the benchmark results, and has concluded that, at this point, they can go either way depending on the workload. He did complain that SLUB can have a higher memory overhead; partial slabs make it better, and further progress can be made on that front. At this point, he concluded, he would not object to removing SLAB. 

Michal Hocko said that SUSE has been using SLUB for some time; it works better in some cases, and worse in others. The biggest reason to make the change, he said, was that SLUB makes debugging problems easier; he suggested just removing SLAB and fixing any remaining problems afterward. Matthew Wilcox said that, in the past, SLUB performed worse with certain database benchmarks, but that problem has since gone away. 

Another attendee asked about SLUB's extra memory overhead; is it something structural, or is it something that can be chipped away at. Babka answered that he was surprised to hear objections about memory overhead. SLUB, it seems, uses about 30% more memory than SLAB to keep track of the memory it manages; he asked whether that translated to significant amount of memory when viewed as an absolute number. 

Much of SLUB's additional overhead, he said, could be seen as a structural problem; SLUB gets its performance by using a lot of per-CPU caches. When Christoph Lameter [introduced SLUB](<https://lwn.net/Articles/229984/>) in 2007, one of his justifications for the addition of another allocator was that SLAB used too much memory for caches. But, Babka said, things have shifted over time. Addressing this memory use would require coming up with another way to get similar performance. 

Pasha Tatashin asked whether per-CPU caching still makes sense in systems with hundreds of cores. Babka answered that some per-CPU caching is needed for scalability, but that there might be ways to make it more effective. 

Concerns about memory usage notwithstanding, the conclusion from the session was that nobody objects to the removal of the SLAB allocator at this point; Babka plans to post a proposal to the mailing lists and see what kind of reaction it gets. Anybody who objects, he said, should be prepared to show a use case or benchmark that regresses with SLUB so that any remaining problems can be addressed. But this removal should not be held back for the sake of a microbenchmark; if there are concrete problems, the community can discuss how to fix them. 

Once that task is complete, he said, it's time to think about what is next. API improvements will become easier once there is only one allocator to change. One idea he had was opt-in, per-CPU caching of array objects which, he said, could improve performance while simultaneously reducing overhead. The ability to allocate in non-maskable interrupt (NMI) context using a per-CPU cache was another idea; there would still be no guarantees that an allocation would succeed, though. That would allow the removal of a BPF-specific allocator. 

Perhaps, he said, the allocator could offer guaranteed allocations with some sort of upper bound, much like mempools do now. That could be useful for tasks like the allocation of [maple-tree](<https://lwn.net/Articles/845507/>) nodes. More generally, he concluded, he would like to find ways to end the reinvention of memory-management functionality outside of the memory-management layer. There are a lot of things being done now that would be better handled in the core memory-management code. 

Wilcox had one problem he would like to see a solution for that he called "dcache poisoning". On a system with a lot of memory and little memory pressure, the directory entry (dentry) cache can grow without bound. This can be an especially big problem with workloads creating a lot of [negative dentries](<https://lwn.net/Articles/890025/>). The kernel will only run the shrinker when there is memory pressure; by the time that happens, cleaning out the dentry cache can take a long time. Andrew Morton described this as a "dentry cache policy decision", but Babka said that the allocator might be a useful part of a solution to this problem. 

Babka closed the session by thanking the attendees and asking them to wish him luck as he proceeds with the SLAB removal.
