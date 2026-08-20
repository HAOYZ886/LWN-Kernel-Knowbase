---
title: NUMA policy and memory types
url: https://lwn.net/Articles/862707/
date: "July 16, 2021"
category: "Memory management-NUMA systems; NUMA"
author: "By Jonathan Corbet July 16, 2021"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 16, 2021

Non-uniform memory access (NUMA) systems have an architecture that attaches memory to "nodes" within the system. CPUs, too, belong to nodes; memory that is attached to the same node as a CPU will be faster to access (from that CPU) than memory on other nodes. This aspect of performance has important implications for programs running on NUMA systems, and the kernel offers a number of ways for user space to optimize their behavior. The NUMA abstraction is now being extended, though, and that is driving a need for new ways of influencing memory allocation; the [multi-preference memory policy](<https://lwn.net/ml/linux-kernel/1626077374-81682-1-git-send-email-feng.tang@intel.com/>) patch set is an attempt to meet that need. 

#### Memory policies

There is no one-size-fits-all allocation policy that yields the best performance for all workloads. If an application can run entirely within a single NUMA node, the best policy is often to allocate all memory on the same node so that all accesses are local (and therefore fast). Larger applications may want to restrict allocations to a subset of the available nodes. For others, though, the best policy may be to distribute allocations across the system so that performance is roughly uniform across all CPUs and overall bandwidth is maximized. As a general rule, the kernel cannot determine what the best policy for any given process might be. 

Thus, it's up to user space to help the kernel with memory-allocation policies; there are two system calls for that purpose. [`set_mempolicy()`](<https://man7.org/linux/man-pages/man2/set_mempolicy.2.html>) sets the default memory policy for the calling thread, while [`mbind()`](<https://man7.org/linux/man-pages/man2/mbind.2.html>) sets policies for specific portions of the calling process's address space. There are a number of options that can be used with either system call. For example, `MPOL_PREFERRED` asks the kernel to allocate memory on a single "preferred" node if possible. A process can, instead, use `MPOL_BIND` to provide a set of nodes that must be used for all memory allocations. `MPOL_INTERLEAVE` asks the kernel to spread allocations, page by page, across the specified set of nodes. And so on. 

These options are generally sufficient for the needs of performance-tuned applications — or, at least, they used to be. But there is a bit of a shift in memory technology underway. The kernel's NUMA support was designed in a world where all memory attached to any given system is the same, and the only difference is the node the memory is attached to. In that world, the only parameter of interest is how local to the CPU any given range of memory is. 

#### The plot thickens

Increasingly, though, systems are being built with multiple types of memory. The most common example at the moment is systems with persistent memory installed; while that memory can be used for long-term storage, it can also be used as a slower form of normal RAM. Persistent memory has the advantage of being relatively cheap; the ability to install lots of it may more than make up for its lower performance for many workloads. Meanwhile, vendors are also working on high-performance memory that is faster than ordinary DRAM, but which is too expensive to use exclusively. 

This presents the kernel (and user space) with a decision that it didn't have to make before: which _type_ of memory should be used to satisfy an allocation request? The solution that has been adopted so far is to organize exotic memory arrays into special, CPU-less NUMA nodes. An application that knows it wants a big chunk of slow memory can set its memory policy to allocate from the appropriate node, and slow memory is what it will get. 

There is one little problem with using the existing NUMA infrastructure this way, though: it doesn't quite fit the problem space. An application that is willing to use slower memory is like an economy-class airline passenger; it is unlikely to be upset if it is given an upgrade at allocation (boarding) time. That application may suggest allocating from a slow-memory node, but allocations should fall back to normal memory if the slow variety is unavailable. 

The semantics of `MPOL_BIND` do not allow for this behavior; it is a more strict regime. If the policy is `MPOL_BIND` and no memory is available on the specified nodes, unfortunate things can happen; these include the unleashing of the out-of-memory killer or the delivery of unexpected `SIGSEGV` signals to the allocating process. The willingness to use slower memory does not generally extend to a willingness to see things collapse in flames if that memory doesn't happen to be available, so it is hard to blame users if they see `MPOL_BIND` as being a little too binding. 

On the other hand, `MPOL_PREFERRED` would appear to be the needed option; it expresses a preference that can be bypassed if the preferred node has no available memory. The problem is that, for reasons known only to the designer of that interface, `MPOL_PREFERRED` only allows the specification of a single preferred node. If the desired memory is distributed across multiple nodes, which is not unlikely, this interface will not allow an application to make use of all of it. 

#### Multi-preference policies

The multi-preference memory policy patch set, which contains work by Feng Tang, Dave Hansen, and Ben Widawsky, seeks to address this problem by adding another option called `MPOL_PREFERRED_MANY`; it behaves like `MPOL_PREFERRED` except that it allows multiple nodes to be specified. Programs using this option can request allocation from a set of nodes offering the desired type of memory, but the kernel can allocate from elsewhere if the desired nodes lack available resources. The patch itself is relatively simple, though it does take a little work to wire up the new option in all of the desired places. 

This work solves the immediate problem, but it does sidestep a relevant question: is the NUMA abstraction the right tool for choosing between different types of memory? Arguments in favor include the facts that it works now, doesn't require a whole new memory-type API, and often matches the actual architecture of the underlying hardware. On the other hand, it does conflate two independent concerns (memory type and locality) and forces user space to work it all out. It feels a little awkward. 

This [is not the first time](<https://lwn.net/Articles/752977/>) that this kind of question has been raised, but the appetite for new APIs remains low. Experience suggests that, as long as the NUMA API can be made to work for memory-type selection, there will not be a lot of pressure to supplement it with something else; the potential benefits probably do not justify the considerable extra cost. So `MPOL_PREFERRED_MANY` is probably the way things will go. The patch set appears to be about ready; this option may appear in a mainline kernel as soon as 5.15.
