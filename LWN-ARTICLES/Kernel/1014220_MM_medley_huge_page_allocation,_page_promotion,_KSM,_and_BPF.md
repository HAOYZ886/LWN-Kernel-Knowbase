---
title: "MM medley: huge page allocation, page promotion, KSM, and BPF"
url: https://lwn.net/Articles/1014220/
date: "March 20, 2025"
category: "BPF-Memory management; Memory management-Huge pages; Memory management-Kernel samepage merging; Memory management-Tiered-memory systems"
author: "By Jonathan Corbet March 20, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 20, 2025

As the [2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) (LSFMM+BPF) approaches, the density of memory-management patches on the mailing lists has increased. Included among those are patches aimed at improving the reliability and performance of huge-page allocation, implementing page promotion on tiered-memory systems, adding a different approach to deduplicating memory, and replacing the BPF memory allocator. Read on for an overview of each. 

#### Reliable huge-page allocation

The use of huge pages — of various sizes — is increasingly important to the overall performance of many, if not most, systems. If memory fragmentation prevents the kernel from allocating huge pages, performance will suffer. While quite a bit of progress in avoiding and fixing fragmentation has been made over the years, the ability to allocate huge pages still decreases over the lifetime of the system. Johannes Weiner has been slowly working on [a patch series](<https://lwn.net/ml/all/20250313210647.1314586-1-hannes@cmpxchg.org>) to improve the situation for a couple of years now; the changes made are relatively small, but their impact may be quite a bit larger. 

The kernel's page allocator tracks the "migration type" of each request; that type describes how easily a given page of data could be moved elsewhere in physical memory if it turns out to be in the way. The [`migratetype` enum](<https://elixir.bootlin.com/linux/v6.13.6/source/include/linux/mmzone.h#L48>) describes these types: whether pages can be moved or easily reclaimed, for example, or whether they are fixed in place. The kernel divides physical memory into contiguous groups called "page blocks", typically holding 512 pages; each page block has an associated migration type, and only compatible allocations are meant to happen from each page block. The intent is to group all of the unmovable pages together, preventing situations where one unmovable page prevents the defragmentation of a large block of memory. 

While the page allocator tries to avoid allocating unmovable pages in movable page blocks, it also tries to avoid performing reclaim and compaction while a caller is requesting memory. An allocation request that starts either direct reclaim or compaction will do a lot of work before returning the asked-for memory, creating latency where it might not be welcome. In current kernels, the page allocator will give up and allocate from a page block with the wrong migration type before trying more expensive remedies. 

Weiner has concluded that those remedies are only more expensive in the short term, though, since misplaced allocations create fragmentation that may last until the system reboots. So his series starts by introducing a new "defrag mode" that can be enabled via a sysctl knob. When this mode is enabled, reclaim and compaction will be attempted before falling back to allocating from a page block with the wrong migration type. Benchmark results shown in [this patch](<https://lwn.net/ml/all/20250313210647.1314586-4-hannes@cmpxchg.org>) demonstrate that, with this mode enabled, the success rate for huge-page allocation no longer degrades over time. 

They also show, though, that the success rate starts at a much lower level than with current kernels and stays there. Two more changes are needed to bring the success rate to a level above what a freshly booted current kernel can achieve, and to keep it there. [The first](<https://lwn.net/ml/all/20250313210647.1314586-5-hannes@cmpxchg.org>) is a small change to instruct the `kswapd` and `kcompactd` kernel threads to try to create page-block-sized chunks of free memory rather than being content with smaller successes; that makes more full page blocks available for allocation as huge pages. The [other change](<https://lwn.net/ml/all/20250313210647.1314586-6-hannes@cmpxchg.org>) raises the bar for the number of free page blocks that those kernel threads aim to create. 

The end result is a much higher success rate for the allocation of transparent huge pages, and a lower allocation latency as well. The results look good, but changes like these have the potential to create performance regressions in other types of workloads. So rather wider testing of this work is likely to be needed before the memory-management community will feel confident about merging it. 

#### Hot-page promotion

In tiered-memory systems — those with memory having different performance characteristics — the kernel must constantly strive to keep the right pages in the right types of memory. Usually, that means placing frequently accessed ("hot") data in the fastest memory that is closest to the workload using it, while less-frequently accessed ("cold") data can be relegated to slower, more distant memory. 

One problem with this approach is that it can be hard to determine how hot a given page is. Detecting memory that has not been accessed for a while is relatively easy, so demoting cold pages to slower memory can be made to work fairly well. The promotion problem, which requires determining which data in slow memory is sufficiently actively used to merit migration to faster memory, is somewhat harder. 

The latest attempt at solving the promotion problem is [this series](<https://lwn.net/ml/all/20250306054532.221138-1-bharata@amd.com>) from Bharata B Rao, which is built on the idea that the number of sources of information on data hotness is growing over time. Classic methods, such as scanning memory and checking the "accessed" bit maintained by the hardware remain. But newer technologies, including AMD's [Instruction Based Sampling](<https://github.com/jlgreathouse/AMD_IBS_Toolkit>) (IBS) and monitoring provided by [CXL](<https://en.wikipedia.org/wiki/Compute_Express_Link>) memory, are coming as well. What is needed is a way to gather and use all of this information to promote the hottest pages to faster memory. 

Rao's patch set adds a new function that can be used by any subsystem that knows something about memory access: 

```
int kpromoted_record_access(u64 pfn, int nid, int src, unsigned long time);
```

This call tells the kernel the memory indicated by the page-frame number `pfn` has just been accessed from the NUMA node indicated by `nid`. The source of the information is provided by `src`, and `time` is the time of the access, in jiffies. Sources are represented by macros like `KPROMOTED_HW_HINTS` for hardware-provided access information, or `KPROMOTED_PGTABLE_SCAN`, for access information obtained by scanning the page tables. The return value is either zero or an error code indicating whether the call was able to record the access information. What the caller is meant to do with that information is not clear; the [IBS driver](<https://lwn.net/ml/all/20250306054532.221138-4-bharata@amd.com>) included in the patch set ignores it. 

Access data is stored in a hash table, keyed by the page-frame number. This storage looks expensive, with a fair amount of data being maintained for each page in the system. The consumer of this data is a new kernel thread called `kpromoted`; it uses an algorithm that Rao describes as ""pretty primitive"" to promote pages that appear to be hot — those that have been accessed a minimum number of times (twice in the current patch) within the last 5ms. 

Much of the code in this series is, by Rao's admission, fairly rudimentary. The purpose at this point is not to provide a polished subsystem for merging; instead, it is to try to get a sense for whether the overall approach is viable. Most of the comments so far have focused on details, but Jonathan Cameron [pointed out](<https://lwn.net/ml/all/20250314152850.00003112@huawei.com/>) that CXL memory providers will be able to provide much of the information that this patch set is working to create by aggregating individual events. Using that data, he suggested, may give good-enough results without needing all of the storage required by Rao's approach. 

#### Simpler KSM

The [kernel samepage merging (KSM) mechanism](<https://docs.kernel.org/admin-guide/mm/ksm.html>) was designed to improve memory utilization by detecting pages of memory with identical contents, then arranging for a single page to be shared while freeing the duplicates. Those pages may be shared across processes that are entirely unaware of each other; the shared memory is (if writable) marked copy-on-write, causing the sharing to be broken should one process write to a shared page. KSM can free a fair amount of memory for other uses, but it has never been used as heavily as might have been imagined. Its page scanning requires CPU time, it requires some fiddly tuning to match the workload, and it raises security concerns since it can be used to determine whether a page with a given content exists elsewhere in the system. 

Mathieu Desnoyers has recently posted [a simplified KSM-like functionality](<https://lwn.net/ml/all/20250228023043.83726-1-mathieu.desnoyers@efficios.com>) that he calls "Synchronous KSM" (SKSM). It is aimed at a specific, seemingly niche use case, though it should be applicable more widely. Desnoyers is working to make run-time code patching more common in user space. Code patching is heavily used in the kernel to take advantage of the optimal instructions for the detected CPU, enable or disable features without the need for run-time tests, enable or disable tracepoints, and more. Desnoyers is looking to bring more of these techniques to user space. 

The problem with run-time code patching is that it breaks the sharing that normally happens with executable code. If many processes are running the same program, they will normally share a single copy of its text; code patching will break that sharing, even if every process patches its code in the same way. That increases memory use and decreases cache locality, taking away the performance that these techniques are meant to bring in the first place. 

SKSM is meant to allow processes, with an explicit opt-in, to restore the sharing of these pages after they have patched them. A call to [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) with the `MADV_MERGE` operation will set up this sharing for the indicated memory range. Significantly, that range is scanned immediately, and any sharing established, in the `madvise()` call itself. There is no kernel thread scanning memory looking for sharing opportunities after the fact. As a result, the overhead imposed by SKSM happens entirely at initialization time; after that, its job is done. 

Linus Torvalds [responded](<https://lwn.net/ml/all/CAHk-=wgedRzDqOLhbOnvziVHZm9jtGOrT4GJEqA9etJDwTQ5Mg@mail.gmail.com/>) to the posting by saying that he had no interest in allowing a second KSM implementation into the kernel, but ""if the feeling is that this can *replace* the current horror that is KSM, I'm a lot more interested"". He [admitted](<https://lwn.net/ml/all/CAHk-=wi5-+P49c3NPeZB_qrNyOtAJS3YadHB0q7J3eZ3UUwrjw@mail.gmail.com/>) that he didn't know who was using KSM now, and worried that it might work well in specific cloud environments and, as a result, cannot be removed. David Hildenbrand [said](<https://lwn.net/ml/all/e110e00f-9032-43ec-808e-45a912065fb0@redhat.com/>) that KSM is mostly used in private clouds, where it can be ""very efficient"", but that its use in public clouds is strongly discouraged due to the associated security concerns. 

As a result, it is not clear that SKSM is going anywhere. The existing KSM functionality, for all its faults, does seem to work for some users, so its removal would not be greeted with universal acclaim. Unless SKSM can somehow be made to fill in for KSM, there may simply not be a place for it in the mainline kernel. 

#### Edging out the BPF allocator

BPF programs can run in almost any context, including within interrupt handlers and even in handlers for non-maskable interrupts (NMIs), where the ability to take locks is severely constrained. That can make allocating memory in those programs challenging. The [BPF memory allocator](<https://lwn.net/Articles/899274/>) was introduced in the 6.1 development cycle as a way for BPF programs to (attempt to) allocate memory in any possible execution context. It works by maintaining its own pool of memory that can be dipped into, without taking locks, when the need arises. This allocator works, but it operates independently of the memory-management subsystem and is counter to the ongoing effort to reduce the number of memory allocators in the kernel. One of the goals [discussed](<https://lwn.net/Articles/974138/>) at the 2024 LSFMM+BPF gathering was the desire to bring allocation for BPF programs back into the memory-management core. 

Alexei Starovoitov has been working on that project; his work so far was [posted in its ninth revision](<https://lwn.net/ml/all/20250222024427.30294-1-alexei.starovoitov@gmail.com>) in late February. It adds a new allocation function: 

```
struct page *try_alloc_pages(int nid, unsigned int order);
```

This function is safe to call in any context. In more constrained contexts, this function will attempt to grab the number of desired pages (indicated by `order`) from the per-CPU free list maintained for the given NUMA node ID (`nid`). Failing that, it will attempt to take the suitable per-zone lock and acquire the memory directly from the buddy allocator. It is only an attempt, though; if the lock is not available, the allocator will fail the request rather than wait for it. As a result, `try_alloc_pages()` has a relatively high likelihood of failure, and callers must be prepared to do without the requested memory. 

There is also a new `free_pages_nolock()` function that can release pages back to the system without taking any locks in contexts where that is not possible. This patch set does not go as far as removing the BPF-specific allocator, though it does provide much of the infrastructure that will be needed to complete that step. 

Andrew Morton initially [resisted](<https://lwn.net/ml/all/20250310190427.32ce3ba9adb3771198fe2a5c@linux-foundation.org>) this work, worrying that the extra maintenance burden it would impose on the core page allocator was not justified by the benefits it would provide. Starovoitov [reminded him](<https://lwn.net/ml/all/CAADnVQJsYcMfn4XjAtgo9gHsiUs-BX-PEyi1oPHy5_gEuWKHFQ@mail.gmail.com>) of last year's discussion, adding that moving this functionality back into the memory-management subsystem would eliminate the wasted memory sitting in the BPF allocator. Slab maintainer Vlastimil Babka [added his support](<https://lwn.net/ml/all/4d75c5a8-a538-4d7d-aaf4-8ecf1d1be6b9@suse.cz/>). Morton resisted no further, and these changes are currently sitting in linux-next, with a probable merge into the mainline happening for the 6.15 release. 

#### Stay tuned

The 2025 LSFMM+BPF gathering begins on March 24 in Montreal, Canada. Many, if not all, of the above topics are likely to see further discussion there. LWN, of course, will be there; keep an eye on these pages for our reporting from the conference.
