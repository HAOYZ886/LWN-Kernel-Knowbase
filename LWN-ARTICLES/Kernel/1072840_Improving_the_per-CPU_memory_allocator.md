---
title: "Improving the per-CPU memory allocator"
url: https://lwn.net/Articles/1072840/
date: "May 19, 2026"
category: "Per-CPU variables"
author: "By Jonathan Corbet May 19, 2026 LSFMM+BPF"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 19, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

There are many places in the kernel where performance can be improved by using per-CPU data. But, as it turns out, the kernel's allocator for per-CPU data has some performance problems of its own. Harry Yoo led a session in the memory-management track of the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) to explore ways to address those problems and accelerate the allocation and initialization of per-CPU data. 

The dynamic per-CPU allocator was [added by Tejun Heo](<https://git.kernel.org/linus/fbf59bc9d74d>) for the 2.6.30 kernel release in 2009; it is built on a per-CPU-data infrastructure whose origin is lost in pre-Git antiquity. Documentation for users of this API is, presumably, in the works and will hopefully show up soon. Allocation of a per-CPU area results in an array of objects, indexed by CPU number, where each CPU's memory lives in a different cache line to avoid contention. A common use for this data is the creation of statistics counters. Each CPU can quickly increment its own counter as needed; when a total is required, it can be obtained by calculating a sum of all the per-CPU values. 

[![\[Harry Yoo\]](https://static.lwn.net/images/conf/2026/lsfmm/HarryYoo-sm.png)](<https://lwn.net/Articles/1072855/>) This being a gathering of memory-management developers, there was particular interest in the per-CPU counters ([`rss_stat`](<https://elixir.bootlin.com/linux/v7.0.6/source/include/linux/mm_types.h#L1266>), stored in each process's `mm_struct` structure) used to keep track of each process's resident set size (RSS). Allocation and freeing of memory is a frequent operation, so managing a process-global RSS counter would be expensive for highly threaded processes. Splitting those counters into a per-CPU array can avoid contention on a global counter but, as was discussed in this session, can lead to other performance problems. 

Yoo began by saying that there are two specific shortcomings with the current per-CPU allocator. One is that it was not designed for scalability; it uses a global lock for the allocation and freeing of memory. It is not uncommon to see multiple CPUs allocating per-CPU memory concurrently, leading to contention on that lock. The other problem is that initializing the per-CPU array can be expensive, especially in cases where there are a lot of CPUs and the data itself is short-lived. There have been a number of proposed solutions, he said, with the result that there are too many ideas floating around but not enough actual progress. 

One of those ideas, he said, was dual-mode per-CPU counters, an idea that was [proposed](<https://lwn.net/ml/linux-mm/20251127233635.4170047-1-krisman@suse.de/>) by Gabriel Krisman Bertazi as a solution to performance problems associated with the RSS counters. Under this proposal, when a set of counters is allocated for a new process, it will be created in a single-threaded mode, since the process itself only has one thread at that point. That greatly reduces the initialization cost. Should the process create a new thread sharing the same address space, the counters are upgraded into the full per-CPU mode. Since many processes never do create threads, this proposal eliminates the initialization cost entirely much of the time. 

An alternative, shown in [this series from Yoo](<https://lwn.net/ml/linux-mm/20250424080755.272925-1-harry.yoo@oracle.com/>), integrates the per-CPU allocator more closely with the slab allocator, and restores the slab destructor operation, which was removed many years ago. That would allow per-CPU objects to be retained in the slab cache and, in particular, would only require the constructor to be called on the initial allocation. This scheme would only work if users of per-CPU objects leave them in a reasonable state when freeing them. Some care would have to be taken with this approach, Yoo said, to limit lock acquisition in destructors to avoid deadlocks. 

Finally, Mathieu Desnoyers (who attended the session via remote link) has [proposed](<https://lwn.net/ml/linux-mm/355143c9-78c7-4da1-9033-5ae6fa50efad@efficios.com/>) using the "per-MM concurrency IDs" (or "mm_cids") for this task. An mm_cid is a virtual CPU ID that is maintained as part of the restartable-sequences subsystem. Unlike a real CPU number, which can be as large as the number of CPUs the system might conceivably support, the mm_cid is bounded by both the number of threads the running process has and the number of CPUs that process is allowed to run on. As a result, it will generally be a much smaller number. A process with four threads on a 256-CPU system might see a CPU number as high as 255, but its maximum mm_cid will be three. See [this article](<https://lwn.net/Articles/885818/>) and [the `rseq()` manual page](<https://lwn.net/Articles/1033957/>) for more details on this feature. 

The core of Desnoyers's proposal, as he described it in the session, is that per-CPU data could be indexed using the mm_cid rather than the actual CPU ID. That would still isolate each CPU's access to the array, but the array itself could be much smaller. Individual entries would be initialized on first use, keeping the initialization cost down. 

Kiryl Shutsemau pointed out that [`get_user_pages_remote()`](<https://elixir.bootlin.com/linux/v7.0.6/source/mm/gup.c#L2547>) can allocate pages on a CPU other than the one it is running on, and Shakeel Butt wondered about remote access to another CPU's counters in general. Shutsemau also raised other cases of remote access, such as when a process is being manipulated with [`ptrace()`](<https://man7.org/linux/man-pages/man2/ptrace.2.html>). In those cases, the code in question is not part of the process it is manipulating, so the mm_cids will not match. Desnoyers said that would be an uncommon case that might be best addressed by adding a separate counter. 

Yoo pushed for a conclusion, saying that some sort of solution was needed in the mainline. Davidlohr Bueso indicated support for the mm_cid idea, asking what problems would prevent it from being adopted. Shutsemau raised the `get_user_pages_remote()` problem again, but said that he would have to look more closely to determine if it is really a problem or not. Suren Baghdasaryan wondered how much extra overhead the mm_cid solution would add in general. 

Another participant pointed out that the RSS counters are known to be imprecise, and that Desnoyers has been circulating [a patch series](<https://lwn.net/ml/all/20260227153730.1556542-1-mathieu.desnoyers@efficios.com/>) to improve them. Some of the proposed solutions to the per-CPU problem are specific to the RSS counter format, he said, suggesting that the mm_cid solution is more general and would be preferable. 

Yoo brought the session to a close by asking, again, what the conclusion should be; the room seemed to be heading toward a consensus to take the mm_cid approach. Butt suggested that Desnoyers should send patches implementing that solution; Desnoyers said that the idea only existed in his head for now, so it would take some time to put together a proper patch set.
