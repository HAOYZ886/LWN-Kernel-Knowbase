---
title: Extending the mempolicy interface for heterogeneous systems
url: https://lwn.net/Articles/973964/
date: "May 18, 2024"
category: "Memory management-Tiered-memory systems"
author: "By Jonathan Corbet May 18, 2024 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 18, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Non-uniform memory access (NUMA) systems are organized with their CPUs grouped into nodes, each of which has memory attached to it. All memory in the system is accessible from all CPUs, but memory attached to the local node is faster. The kernel's [memory-policy ("mempolicy") interface](<https://man7.org/linux/man-pages/man2/set_mempolicy.2.html>) allows threads to inform the kernel about how they would like their memory placed to get the best performance. In recent years, the NUMA concept has been extended to support the management of different types of memory in a system, pushing the limits of the mempolicy subsystem. In a remotely presented session at the [2024 Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Gregory Price discussed the ways in which the kernel's memory-policy support should evolve to handle today's more-complex systems. 

Heterogeneous-memory systems may seem like exotic beasts, Price began, but they are actually common; even a simple two-socket server, with its two banks of memory with different access characteristics, is a heterogeneous-memory system. On such systems, developers have to think about where their tasks run, or performance will suffer. Future systems will be worse, though; they will be "a monstrosity", equipped with ordinary DRAM (at various distances), [CXL memory](<https://en.wikipedia.org/wiki/Compute_Express_Link>), [high-bandwidth memory](<https://en.wikipedia.org/wiki/High_Bandwidth_Memory>), and more. The kernel's mempolicy API was not designed for this kind of system — or even today's basic two-socket system, he said. 

Memory tiering has been a frequent topic of discussion at LSFMM+BPF for some years now, and memory policy clearly will be a part of the tiering solution, but tiering and mempolicy are aimed at slightly different problems. The tiering discussion is all about memory _movement_ between different memory tiers, while the mempolicy interface is about _allocation_. The former is focused on migration, while the latter is about node selection. In a perfect world, the kernel would always place memory allocations perfectly, but we do not live in that world. Allocations will be wrong, or usage patterns will change over time. Thus, he said, tiering is useful and necessary — but so is better allocation policy. 

In current systems, every thread can have its own memory policy; that policy can even be different for each virtual-memory area in the thread. There are four policy types available to control where allocations are placed: default to the local node, allocate on a set of preferred nodes, interleave across a set of nodes in a round-robin fashion, and weighted interleaving. 

The last option, [weighted interleaving](<https://lwn.net/Articles/948037/>), was added for the 6.9 kernel. It is controlled with a set of global weights managed via sysfs. The administrator can use these weights to try to obtain optimal bandwidth use across all memory interconnects; putting some frequently used data in slower memory can improve performance overall if it keeps all of the interconnects fully busy. Weighted interleaving can thus improve throughput, but can also complicate the latency story. This mechanism is sufficient for simple tasks, and a number of useful lessons have been learned from its implementation. 

#### Lessons learned

One of those lessons is simply that the kernel's memory-policy features have not kept up with the evolution of the computing environment in which they run. Consider, he said, a single-socket system running with attached CXL memory, which is slower than DRAM. A streaming benchmark will run 78% slower on that system than on a machine with DRAM only. But, with a proper, task-wide weighted-interleaving policy, that benchmark will run somewhere between 6% slower and 4% _faster_. That is better, "but it still sucks". It is possible to get good results on such systems, but processes are forced to be NUMA-aware to get those results. 

The current mechanism is built around the idea that either the administrator or some sort of daemon will manage the weights used for interleaving. He has [an RFC patch](<https://lwn.net/ml/linux-kernel/20240220202529.2365-1-gregory.price@memverge.com/>) circulating to do this automatically using information from the system's heterogeneous memory attribute table (HMAT), but that is not an easy thing to do, especially in systems where memory hotplugging is in use, on complex NUMA systems, or on systems with other types of complex memory topologies. Task-local weights can help, but that feature was dropped out of the patch set merged for 6.9, because it needs some new system calls; he has [another RFC patch set](<https://lwn.net/Articles/969379/>) out there that adds them. 

While the current memory-policy API can be made to work, it is unwieldy at best on large NUMA systems. Sub-NUMA clustering (a recent hardware feature that partitions NUMA zones into smaller sub-zones) is hard to use well with this API. In general the number of nodes showing up on systems is growing, but that makes the system as a whole harder to reason about, he said. 

The memory-policy interface is entirely focused on the currently running task; there is no way for one thread to change another's policies. Within the memory-management subsystem, policy changes require a level of access to the virtual-memory areas (VMAs) that will be painful to extend. The current design is not without its advantages; it allows the implementation of memory policies to be lockless in the allocation paths. Widening access without hurting performance will require some significant refactoring and movement toward the use of read-copy-update (RCU). Memory policies also have complex interactions with control groups, and must not violate any restrictions imposed by control groups. 

Michal Hocko asked how VMA-level manipulation could be implemented without creating other problems; Price answered that there is a patch for a new system call (`[process_mbind()](<https://lwn.net/ml/linux-kernel/ZV5zGROLefrsEcHJ@r13-u19.micron.com/>)`) circulating now. Hocko answered that the patch "is not wrong", but that it is complicated and has security implications. 

David Hildenbrand asked whether Price was thinking that a system would run a process that would be adjusting the VMAs of others, or would applications opt into some sort of management scheme? Price answered that allowing the first case is the important part of this work; other types of mechanisms can come later if need be. There is no agreement on the existing work yet, though, so there will be changes to those patches, including trying to make more use of existing system calls (like [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>)) when it makes sense. 

Liam Howlett asked how memory policies would be affected if the scheduler moves a task elsewhere in the system. This is a problem that has been talked about a lot, Price answered. One of the reasons for the global interleaving weights is that they ease the problem of dealing with process migration. That is also part of why the other system calls have been pushed back. 

#### Proposals

Price concluded with a quick look at what is being proposed for the memory-policy subsystem. It would be good to get to the point where a process running with a reasonable policy would get performance close to what can be had by explicitly binding memory to nodes. That involves finding ways to _not_ interleave memory for data that is not driving the system's memory-bandwidth use. The plan is to implement `process_mbind()` in some form; it will use the pidfd abstraction and be analogous to [`process_madvise()`](<https://lwn.net/Articles/810076/>). This mechanism could be seen as a sort of crude tiering solution that would be useful to job-scheduling systems. 

There is also a wish to improve how [`mbind()`](<https://man7.org/linux/man-pages/man2/mbind.2.html>) performs memory migration. Currently, bound memory will only be migrated if a node is removed from the allowed set. But if a process is set up for interleaving, and a new node is added, there will be no migration to rebalance the allocations. That would be a nice feature to have, but implementing it could be expensive, he said. If it can be done, though, he would like to see redistribution in the interleaved case — and the configured weights should be applied when this happens. 

Finally he asked whether memory policies should be integrated with control groups. That could be awkward, since memory policies are lockless, while control groups are not. Hocko was skeptical, saying that control groups are all about hierarchies, and he does not see a way to define a reasonable hierarchical policy. Price said, though, that control-group integration would ease the management of sets of policies, and simplify the handling of migration. But he acknowledged that this idea has not found any sort of consensus; he will continue looking for solutions.
