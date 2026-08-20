---
title: "Transparent huge pages, NUMA locality, and performance regressions"
url: https://lwn.net/Articles/787434/
date: "May 6, 2019"
category: "Memory management-Huge pages"
author: "By Jonathan Corbet May 6, 2019 LSFMM"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 6, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Sometimes, the kernel's no-regression rule may not have the desired result. Andrea Arcangeli led a session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit to make the point that the recent reversion of a fix after a performance regression was reported has led to worse performance overall — with, as is his wont, a lot of technical information to back up that point. With a wider understanding of what is at stake here, he hopes, the reversion can itself be reverted. 

It started with a performance problem reported in the 4.2 kernel, when Alex Williamson encountered pathological behavior around transparent huge pages (THP). When using THP in large virtual memory areas under [VFIO](<https://www.kernel.org/doc/Documentation/vfio.txt>), a large slowdown would result. After some work, the problem was figured out and a fix was applied to the RHEL kernel in August 2018. A little later, the same [![\[Andrea
Arcangeli\]](https://static.lwn.net/images/conf/2019/lsfmm/AndreaArcangeli2-sm.jpg)](<https://lwn.net/Articles/787436/>) problem was reported in an SLES 4.12 kernel; the same fix solved it there. At that point, the fix [was committed upstream](<https://git.kernel.org/linus/ac5b2c18911f>) as being of general use. But that fix ended up being [reverted](<https://git.kernel.org/linus/2f0799a0ffc>) in December after automated testing revealed a slowdown in one benchmark; while the fix had greatly improved fairness, it did alter performance in this case. Large amounts of email traffic ensued; Arcangeli hopes that it will soon be possible to apply the fix again. 

The underlying problem is that, if the system is configured to always enable memory compaction and a huge page allocation is requested, the page allocator will refuse to allocate pages on remote nodes. It behaves as if the program had been explicitly bound to the current node, which was never the intended result. The reasoning that led to this behavior is that it is better to allocate local 4KB pages than remote huge pages — an idea that Arcangeli is not fully in agreement with. But the kernel goes beyond that in this situation, refusing to allocate _any_ pages on remote nodes and potentially forcing the local node deeply into swap. The (since reverted) fix avoids this situation by allowing the allocation of remote pages (both huge and small) when local pages are not available. In a quick demo, Arcangeli showed how that can yield better performance and less local-node swapping. 

Even so, he said, the allocation heuristic is not what it should be: it tries to allocate huge pages across the system first before falling back to 4KB pages locally. The correct order would be to first attempt to allocate a huge page locally, then to try a 4KB page locally. Failing that, the next thing to try should be a huge page at the closest remote node, followed by a 4K page at that node, then a huge page at a more distant node, and so on. In other words, the allocator should follow something like the current zone-list heuristic, but attempting allocations at multiple sizes at each step. 

Doing that properly may require changing the `alloc_pages()` interface to make it smarter; he suggested something like `alloc_pages_multi_order()` that would accept a bitmask of the acceptable allocation sizes. It would try the largest first, but then fall back to the smaller size(s) if need be. Naturally, it would return the size of the actual allocation to the caller. 

There was a bit of discussion on how such an interface might really work. Michal Hocko said, though, that the real problem is that the compaction setting has other, surprising implications: it can result in a failure to allocate huge pages in situations where they would help. Perhaps the solution is just to be a bit more clever in how this setting is handled, in which case it should just be done. 

Meanwhile, Arcangeli's intent is to get the original fix reapplied; he has [posted a patch set](<https://lwn.net/ml/linux-kernel/20190503223146.2312-1-aarcange@redhat.com/>) doing so with a bunch of backup information. That seems like the right thing to do; two enterprise distributions are currently carrying out-of-tree patches to address this issue. Once that little problem has been resolved, work can begin in earnest on coming up with the best way to allocate huge pages in NUMA environments.
