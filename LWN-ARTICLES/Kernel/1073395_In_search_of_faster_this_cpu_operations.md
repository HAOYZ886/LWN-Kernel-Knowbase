---
title: In search of faster this_cpu operations
url: https://lwn.net/Articles/1073395/
date: "May 19, 2026"
category: "Per-CPU variables"
author: "By Jonathan Corbet May 19, 2026 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 19, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

The kernel's [this_cpu operations](<https://docs.kernel.org/core-api/this_cpu_ops.html>) are meant to speed access to per-CPU variables. They are more optimal on some CPUs than others, though. During a memory-management-track session at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Yang Shi proposed a fundamental, and somewhat controversial, change to how these operations work in order to provide better performance on a wider range of architectures. 

[![\[Yang Shi\]](https://static.lwn.net/images/conf/2026/lsfmm/YangShi-sm.png)](<https://lwn.net/Articles/1073398/>) Per-CPU variables are organized as an array, indexed by CPU number, with each CPU's data being placed in a separate cache line; that allows CPUs to access their private data without locking overhead or contention from other CPUs. On the x86 architecture, a per-CPU access is implemented by prefixing the relevant instruction with a segment register, creating a single-instruction operation that executes atomically. Other architectures, including Arm, lack segment registers and, as a result, have a more complicated implementation. In particular, the calculation of the address and the access of the data behind that address must be done separately, turning a per-CPU access into a multi-instruction sequence. 

That is a significant difference. If a multi-instruction sequence is preempted partway through, the newly running thread could access the same variable, leading to all sorts of unpleasant behavior. Migration of the thread could lead to cross-CPU access, which is also undesirable. To prevent these scenarios, the this_cpu operations must disable preemption on the affected architectures, which hurts performance. 

Shi's proposal is to reimplement per-CPU variables so that any given variable has the same address on every CPU, using per-CPU page tables to make that work. That would eliminate the index calculation and preemption would no longer be a problem. The only problem with this scheme is that it would break the `per_cpu_ptr()` macro that is used to initialize data across all CPUs. To solve that problem, per-CPU variables would be mapped twice. The existing global mapping would be retained and used for initialization; a second mapping would be specific to each CPU. 

Jason Gunthorpe pointed out that, in the past, Linus Torvalds has been strongly opposed to using per-CPU page tables in this way. If his concerns are not addressed, Gunthorpe said, this work will not go far. The objection seems to be based on the difficulty of correctly managing translation lookaside buffer (TLB) entries associated with those addresses; there was some discussion about whether that situation has improved with more recent architecture revisions. It was also pointed out that, while this scheme would eliminate the need to disable preemption in some situations, any sort of update involving more than one instruction would still need to be executed with preemption disabled. 

Shi continued, saying that the new implementation does have a cost in the form of higher address-space usage; that cost appears to be small, though. The per-CPU page tables will need to occupy physical memory; he said that cost is about 2MB on a 160-core machine. Some extra page-table operations are needed for the allocation and freeing of per-CPU variables. There may also be a need to allocate a dedicated range of virtual address space, which could be a problem on 32-bit machines. 

Performance benchmarks were run on a 160-core Arm system; the key kernel-build benchmark showed a 13-18% reduction in system time, and ran in 3-7% less wall-clock time. The [stress-ng benchmarks](<https://wiki.ubuntu.com/Kernel/Reference/stress-ng>) showed significant improvements as well. Brendan Jackman expressed surprise at how large those numbers were; avoiding the need to disable preemption does not seem like enough to explain the difference. Some more investigation into the cause of the speedups is indicated, he said. 

As time ran out, Ryan Roberts wondered if the [in-kernel restartable-sequences work](<https://lwn.net/ml/all/20260223163843.GR1282955@noisy.programming.kicks-ass.net/>) from Peter Zijlstra might be an alternative solution to this problem; Shi was not familiar enough with that work to say. Shi did say that the per-CPU page tables could, in the future, also be used for replication of the kernel text across NUMA nodes, providing local access across the system. Jackman said that they could make his proposed "mermap" (which was [discussed](<https://lwn.net/Articles/1072367/>) earlier in the conference) work better as well.
