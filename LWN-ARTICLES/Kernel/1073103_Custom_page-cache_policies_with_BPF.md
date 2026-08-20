---
title: "Custom page-cache policies with BPF"
url: https://lwn.net/Articles/1073103/
date: "May 22, 2026"
category: "BPF-Memory management; Memory management-Page cache"
author: "By Jonathan Corbet May 22, 2026 LSFMM+BPF"
---

By **Jonathan Corbet**  
May 22, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

The kernel's page cache is charged with maintaining pages (or, more correctly, [folios](<https://lwn.net/Articles/1064861/>)) containing copies of data from files in the filesystem; its performance has a big effect on the performance of the system as a whole. One of the key decisions the kernel must make is when to evict folios from the page cache. At the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Tal Zussman ran a memory-management-track session on how the page cache could be better customized for specific workloads. It will not be much of a spoiler to say that it involves BPF. 

Eviction from the page cache can be managed by either the traditional least-recently-used (LRU) algorithm or the multi-generational LRU; these subsystems were [discussed in detail](<https://lwn.net/Articles/1072866/>) earlier in the Summit. Zussman started by saying that there are workloads that are not served well by either of those options. As an example, he mentioned an unnamed financial database that performs a large number of small, performance-sensitive queries; there is also a set of lower-priority analytical tasks. The database will fill the page cache while handling the high-priority work but, when an analytical scan starts up, most of the data needed to satisfy those queries is pushed out. The next such query then ends up thrashing all of that data back in. The current page cache, he said, is not flexible enough to address this problem; it is leaving performance on the table. 

Vlastimil Babka asked why the kernel's access-twice heuristic does not prevent this scenario. That heuristic marks data as inactive when it is first added to the LRU lists; the second access causes it to be marked as active. Inactive data is evicted first, so this heuristic keeps single-use data from pushing out more useful pages. Zussman answered that there are often multiple scans happening at the same time, fooling that heuristic, but the real problem is that the page cache lacks awareness of how the application accesses data. 

There are three ways that this problem might be fixed, he said. One would be to change the LRU policy implemented by the kernel, which is hard to do and would probably regress other workloads. The existing hint interfaces (such as [`posix_fadvise()`](<https://man7.org/linux/man-pages/man2/posix_fadvise.2.html>)) are not able to affect eviction policy in useful ways for many workloads. The second way to solve the problem is application-level caching, which has downsides of its own, including duplicating the caching done by the page cache. Using direct I/O can address the duplication problem, but it requires reimplementing a lot of functionality that the kernel is already providing. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

So, he said, there needs to be a way to fundamentally change the underlying policy. The [sched_ext](<https://lwn.net/Articles/974387/>) subsystem makes that possible for CPU scheduling; he is proposing a similar feature, called [cache_ext](<https://github.com/cache-ext/cache_ext>), for the page cache. It would allow page-cache policies to be loaded (as a BPF program) from user space, with no kernel changes, and attached to control groups. It is implemented as a [struct_ops program](<https://lwn.net/Articles/811631/>) with callbacks to inform the program when folios are added to or removed from the page cache, and when they are accessed. Another callback requests the program to evict a number of folios from the page cache. 

Shakeel Butt asked whether this interface would be general enough to manage all of memory, not just the page cache; Zussman answered that he is focused on file-backed memory for now, but the interface could probably be extended. The set of hooks needed for anonymous pages would likely be different, though. 

Policies, he continued, operate on eviction lists maintained by the BPF program. When a folio is added to the page cache, the program picks a list to add that folio to; at eviction time, the program chooses from whichever list it thinks is best. It is a simplistic structure, he said, but most policies can be implemented using these lists. An audience member asked if the lists could be managed in a [BPF arena](<https://lwn.net/Articles/1019885/>); Zussman said that might be possible, but that limitations within BPF make it hard. 

For the financial workload described above, user space would inform the cache_ext program which applications are performing scans by storing their process IDs in a BPF map. When a folio is added to the page cache, the program checks to see if it is being added on behalf of one of those scan applications; if so, the folio is put onto a special list that is targeted first at eviction time. For this application, he said, this policy produced a 70% increase in query throughput and a big reduction in tail latency. 

David Hildenbrand asked whether there is an access callback that can recognize a high-priority task and move the relevant folios to a new list; Zussman said that could be implemented, but is not being done now. Kiryl Shutsemau said that turning to BPF is an overreaction to the problem, and suggested that the focus should be on creating better kernel interfaces instead. Improvements to `posix_fadvise()`, perhaps, could set the "don't cache" flag on folios that should be evicted quickly. Matthew Wilcox, though, expressed regret that this flag, which consumes a scarce page flag, had ever been added. Had cache_ext existed before, he said, that flag would never have been necessary. 

Liam Howlett expressed some dismay that cache_ext uses policies attached to control groups; Zussman said that the LRU lists are already maintained for each control group, so that is a natural place to apply the policy. In this way, different groups can have different policies. Howlett said that, if the workload changes, the application is still locked into the old policy unless it is moved to a new control group; Zussman said that the policy can be changed at any time. 

Babka asked whether the eviction hook runs when reclaim is done globally, or when it happens at the control-group level; the answer was the latter. Brendan Jackman asked why the policy was being applied at the page-cache level rather than, say, to the inode cache? Zussman said that there may be a place for policies at the inode level, but that would be a much higher-level policy. 

Hildenbrand asked how a control group would be transitioned to a new policy; the answer is that switching is a destructive operation — the lists that had been constructed by the old policy would be lost. It is possible to export some knobs to tune policies, which could avoid the need to replace them entirely much of the time. Hildenbrand said that, at times, the memory-management subsystem will remove specific folios from the LRU lists for a while, and asked if the policies could handle that. Zussman answered that any metadata stored in BPF maps would persist in that situation, but the list information would be lost. 

The last question came from Shutsemau, who asked whether the workload will be expected to tag page-cache requests somehow for the benefit of the policy. The answer, Zussman said, depends on what the goal is. If the intent is to create a generic policy that does not understand the behavior of specific applications, there will be no use for tagging. A more application-aware policy, though, may want information from the application about what it is doing. 

Zussman has [posted the slides](<https://talzussman.com/slides/cache_ext_lsfmm2026.pdf>) from this session.
