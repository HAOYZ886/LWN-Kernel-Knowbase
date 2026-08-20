---
title: "Kernel-text replication on NUMA systems"
url: https://lwn.net/Articles/956900/
date: "January 5, 2024"
category: "Memory management-NUMA systems; NUMA"
author: "By Jonathan Corbet January 5, 2024"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 5, 2024

Kernel developers often go out of their way to reduce the memory used by the kernel itself; that memory is not available for the workloads that people are actually interested in running on their systems. Lower memory usage also tends to lead to better performance overall. But there are times when the expenditure of some extra memory can make the system faster. The replication of the kernel's text (executable code) and read-only data across a NUMA system may be a case in point; patch sets have been posted adding that capability to two architectures. 

Non-uniform memory access (NUMA) systems are divided into nodes, each of which normally contains a number of CPUs and some memory. All of the memory in the system is addressable from any CPU, but access to memory attached to the local node will be significantly faster than access to memory on remote nodes. For this reason, the kernel goes out of its way to try to keep workloads and their memory on the same nodes, and there is an extensive set of system calls allowing applications to control where in the system their memory is placed. 

But how should the system place data that is needed by _all_ of the nodes in the system? For writable data, there is usually little to be done; there must be a single copy somewhere. At best, that data can be spread across all of the nodes so that access times are relatively uniform everywhere. The same can be done with read-only data — such as the kernel's text — so that workloads will perform similarly regardless of which node they land on, though (as we'll see) not all architectures do that. But the read-only nature of that data suggests another approach as well: make a copy of that data on each node, so that every node has fast access to it. For frequently accessed data like the kernel text, the result should be improved performance, at the cost of the extra memory needed for the duplicate copies. 

In May 2023, Russell King posted [an implementation of kernel-text replication for the arm64 architecture](<https://lwn.net/ml/linux-doc/ZHYCUVa8fzmB4XZV@shell.armlinux.org.uk/>). The fact that this work showed up on that architecture is indicative of how CPU design has changed; NUMA architectures were once restricted to high-end servers, but now they are just as likely to show up in a phone. As King notes, the arm64 architecture currently places the kernel text on a single node, leading to performance disparities across the system. Copying the kernel text to each node will give each node local access, thus improving performance. 

Each node's copy of the text will, naturally, be located at a different physical address. That difference has to be hidden, though; as tasks migrate between nodes, they need to find the kernel at the same virtual addresses. King manages that problem by changing the placement of the kernel text (and read-only data as well) so that it is covered by its own top-level page-table directory. That directory can easily be set differently for each node, resulting in a mapping where the kernel text sits at the same virtual address everywhere, even though its physical location varies. 

Once that is set up, it's mostly just a matter of allocating more memory, mapping it properly, and copying the kernel text into it. Except, of course, there are details. For example, the kernel text is not truly read-only; there are times (such as when a tracepoint is enabled, a module is loaded, or a BPF program is installed) where the kernel patches its own text at run time. All of those operations must now be replicated across all copies of the kernel text. There are some outstanding problems as well that King noted in his posting. Kernel-text replication is incompatible with kernel address-space layout randomization (KASLR), so that security mechanism must be disabled. Text replication also breaks [KASAN](<https://docs.kernel.org/dev-tools/kasan.html>); King asked for help fixing it, but that help does not appear to have materialized. 

In response that posting, Ard Biesheuvel [said](<https://lwn.net/ml/linux-doc/CAMj1kXHn0oho_CZMSc5N1updfdZDq+3VAfzw8kZqzzpTSgkXew@mail.gmail.com/>) that ""the proposed approach is sound, but it is rather intrusive"". He noted that, if and when [his patches](<https://lwn.net/ml/linux-kernel/20230307140522.2311461-1-ardb@kernel.org/>) adding support for LPA2 (large physical addresses for arm64) is merged, there will be another point of incompatibility with text replication. As an alternative approach, he suggested [protected KVM (pKVM)](<https://source.android.com/docs/core/virtualization/security>) could be used to run a separate kernel in each node, with the guest-level page tables being used to map the replicated kernel text. King [responded favorably](<https://lwn.net/ml/linux-doc/ZJW7kvWqLVZV4KVr@shell.armlinux.org.uk/>) to the idea but, when he posted [a second revision of the replication patches](<https://lwn.net/ml/linux-doc/ZMKNYEkM7YnrDtOt@shell.armlinux.org.uk/>) in July, it kept the same approach as the first version. 

That posting received almost no review comments, and the work appears to have stalled. That is unfortunate since, as noted in the original posting, there can be real benefits to it: 

> Needless to say, the performance results from kernel text replication are workload specific, but appear to show a gain of between 6% and 17% for database-centric like workloads. When combined with userspace awareness of NUMA, this can result in a gain of over 50%. 

King's patches have not been reposted since July. The idea of replication is alive and well, though, as can be seen in [this series](<https://lwn.net/ml/linux-mm/20231228131056.602411-1-artem.kuzin@huawei.com/>), posted by Artem Kuzin, adding replication support to the x86 platform. The cover letter makes reference to King's work and expresses hopes that it will be possible to push both projects forward together. This rather larger patch set does not build directly on what came before, though. 

The arm64 architecture cleanly separates the kernel and user-space page tables, making replication a bit easier to implement; x86, instead, lacks that separation, with the result that the implementation is a bit more complex. Rather than limit node-specific page-table directories to the top level, this series has to create node-specific PUD and PMD (the next two levels down on four-level systems) directories as well. This approach is, however, able to support KASLR and KASAN (though KASAN evidently does not yet work with five-level page tables). 

Here, too, benchmark results would seem to indicate that significant performance improvements are possible. 

This series has seen even less review attention than the arm64 patches. The fact that this work is scrupulously undocumented may be a part of that, but it also was only posted on December 28, meaning that most developers have not yet had a chance to take a good look at it. Once the development community fully recovers from the holidays, discussion on this work might pick up. 

In theory, this idea should not be particularly controversial. The amount of extra memory needed to replicate the kernel is not huge, especially in the context of larger NUMA systems, and the performance benefits look to be large. This change, on either architecture, does require a fair amount of intrusive patching of the core kernel, though, and can add complexity to the implementation of other important features as well. Whether this work finds its way into the mainline will ultimately depend on whether that added complexity is seen as being worthwhile given the benefits it brings.
