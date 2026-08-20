---
title: "Page allocation for address-space isolation"
url: https://lwn.net/Articles/1016013/
date: "April 3, 2025"
category: "Memory management-Address-space isolation"
author: "By Jonathan Corbet April 3, 2025 LSFMM+BPF"
---

By **Jonathan Corbet**  
April 3, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Address-space isolation may well be, as Brendan Jackman said at the beginning of his memory-management-track session at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit, ""some security bullshit"". But it also holds the potential to protect the kernel from a wide range of vulnerabilities, both known and unknown, while reducing the impact of existing mitigations. Implementing address-space isolation with reasonable performance, though, is going to require some significant changes. Jackman was there to get feedback from the memory-management community on how those changes should be implemented. 

The core idea behind address-space isolation (last [covered here](<https://lwn.net/Articles/1014440/>) in March), he began, is to run as much kernel code as possible in an address space where sensitive data is unmapped, and thus invisible to speculative-execution vulnerabilities. It is like the kernel page-table isolation that was [introduced](<https://lwn.net/Articles/741878/>) in response to the Meltdown hardware vulnerability, but with a higher degree of protection. Kernel page-table isolation created a new address space with most of the kernel removed; the new work adds a restricted address-space that has holes in it where only the sensitive data has been removed. 

[![\[Brendan Jackman\]](https://static.lwn.net/images/conf/2025/lsfmm/BrendanJackman-sm.png)](<https://lwn.net/Articles/1016018/>) The address-space isolation patches are deployed on a significant subset of Google's fleet, he said. Their current (public) form can be seen in [this patch set](<https://lwn.net/ml/all/20250110-asi-rfc-v2-v2-0-8419288bc805@google.com/>) posted in January. This version adds protection from bare-metal attackers, while previous versions had only protected the kernel from virtual machines. There are still two blockers that need to be addressed though. One is a better design for page allocation — the intended topic for this session — while the other is a 70% degradation in file-I/O performance. 

In order for the kernel to unmap memory containing sensitive data, it needs to know where that memory is, so kernel code must indicate sensitivity at allocation time by way of a new `__GFP_SENSITIVE` flag. There are some performance considerations here; for example, mapping pages may require first zeroing them, since they may have previously contained sensitive data. He would also like to avoid fragmenting the kernel's direct map if possible. Mike Rapoport, who has [analyzed the cost of direct-map fragmentation](<https://lwn.net/Articles/931406/>), said that it is best avoided if possible, but is not that critical. 

Avoiding fragmentation, Jackman continued, requires grouping nonsensitive pages together in physical memory. He also preallocates page tables for restricted data down to the PMD (2MB) level, and maps or unmaps entire 2MB page blocks at one time. That helps to minimize fragmentation and translation lookaside buffer (TLB) invalidations. 

The patch set adds two new migration types to distinguish sensitive and non-sensitive data, and a new constraint that disallows the allocation of pages across the two sensitivity types. 

There are some challenges that come with unmapping page blocks while allocating memory. The unmapping requires a TLB invalidation, but that cannot be done while the zone lock (needed to allocate the page block) is held. The invalidation must be done, though, before other CPUs are allowed to see the block as being sensitive. So the current code allocates the entire page block, even if only one page is needed, releases the zone lock, performs the invalidation, then reacquires the zone lock. After that, the memory can be marked sensitive and any unneeded pages can be freed. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

That technique works, but leads to a possible worrisome scenario. If all CPUs on the system decide to allocate a sensitive page at the same time, they will all end up doing the above dance, and they will all hammer each other with TLB invalidations. Jackman said that he is not sure that this case is worth optimizing for, but Matthew Wilcox said that database workloads could possibly act in just that way. 

Mapping a block while allocating is easier, Jackman said; it is just a matter of populating the page tables and changing the migration type of the affected memory. It is essentially a normal case of migration-type fallback. But pages that might have held sensitive data have to be zeroed to prevent the possible exposure of that data; he wondered if the allocator should just zero pages unconditionally. The cost of doing so, he said, is not that bad. Jason Gunthorpe said, though, that zeroing can indeed be painful on systems with slower memory, and Suren Baghdasaryan said that there had once been a maple-tree performance regression caused by page zeroing. 

If the zeroing overhead is too much, Jackman continued, then the allocator will have to repeat the unmapping dance described above, or handle zeroing one page at a time, using a page flag. Somebody asked what the performance of the allocator was at Google; Jackman said that it worked well, but the version of address-space isolation running there does things differently than the version that has been posted for upstream consideration. 

Wilcox asked if the kernel's Spectre mitigations can be safely disabled once address-space isolation is in use. For now, Jackman said, the isolation only protects user-space pages; the task of marking kernel allocations for sensitivity is far from complete. Once that has been done, it should be possible to turn off the mitigations, and to never need another one again. The mitigations are off at Google, and the patch yields a performance gain overall. 

Since he had some time at the end of the session, Jackman launched into the problem of the file-I/O performance regression caused by address-space isolation. The problem is that all file pages are marked sensitive within the kernel, so every read causes a fault and an address-space transition. It is pointless to protect pages that a process is about to read anyway, but the page cache as a whole cannot be marked non-sensitive since it likely contains data that any given process cannot access. Earlier versions of the patch set had a separate "local nonsensitive" marker for data that processes could leak to themselves but, even with that, the kernel does not know at allocation time where file pages should be mapped. 

Thus, he said, the kernel needs a process-local mapping for file pages. One solution would be to map entire files when a process opens them; that is relatively easy, but it is harder to know when to unmap file data. A process can lose access to a file in a number of ways; a security module might change its mind, or [fanotify](<https://man7.org/linux/man-pages/man7/fanotify.7.html>) permission events can revoke access, for example. There must also be action taken when file pages are removed from the cache, either through reclaim or because a file is truncated. 

The alternative, he said, is ephemeral, per-CPU mappings that are in place only as long as the operation is ongoing. Once the operation completes, the page tables would be torn down right away, but the TLB flush could be deferred to minimize the performance impact. 

At that point, the session was truly out of time and the discussion ended with no conclusions on the file-I/O problem. 

Jackman has posted [the slides from this session](<https://docs.google.com/presentation/d/1waibhMBXhfJ2qVEz8KtXop9MZ6UyjlWmK71i0WIH7CY/edit#slide=id.p>).
