---
title: Extending restartable sequences with virtual CPU IDs
url: https://lwn.net/Articles/885818/
date: "February 28, 2022"
category: Restartable sequences
author: "By Jonathan Corbet February 28, 2022"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 28, 2022

Restartable sequences, a Linux kernel feature that facilitates the writing of lockless, per-CPU code in user space, has been around for some years, but it only just [received support in the GNU C Library](<https://lwn.net/Articles/883104/>) this month. Now that this barrier has been crossed, it would seem that the time has come to start adding features. Mathieu Desnoyers has responded to this challenge with [a patch set](<https://lwn.net/ml/linux-kernel/20220218210633.23345-1-mathieu.desnoyers@efficios.com/>) adding an extension mechanism and a new "virtual CPU ID" feature. 

See the above-linked article for an overview of how restartable sequences work. As a reminder, any thread using restartable sequences must first make use of the `rseq()` system call to register a special structure with the kernel. That structure is used to point to the `rseq_cs` structure describing the current critical section (if any); the kernel also ensures that it contains the ID number of the current CPU whenever the thread is running. Consistent with the pattern used in many relatively recent system calls, `rseq()` requires the caller to also provide the size of the `rseq` structure being passed in. 

That length parameter exists to support future extensions to the system call. New features will generally require new data, increasing the size of the `rseq` structure. By looking at the size passed by user space, the kernel can tell which version of the `rseq()` API the calling process expects. When carefully used, this mechanism allows existing system calls to be extended in a way that preserves compatibility with older programs. 

That still leaves an open question for programs that need to discover which API version they are dealing with as a way of knowing which features are available. One possibility is to invoke the system call with the most recent version of the structure and fall back to an earlier version if the call fails. Another is to simply have the kernel say which structure size it is prepared to accept. The `rseq()` patches take the latter approach, making the maximum accepted structure size available via [`getauxval()`](<https://man7.org/linux/man-pages/man3/getauxval.3.html>). 

Having added this extension mechanism, the patch set goes on to add two extensions without actually using it. These extensions add two 32-bit values to `struct rseq`, which does extend its length. But, due to the way that the structure was defined (with 32-byte alignment), it will already have a 32-byte allocated size, even though the (pre-extension) structure only required 20 bytes. That said, user space will still be able to tell whether the new values are supported by looking at the return value from `getauxval()`. Since the new value (`AT_RSEQ_FEATURE_SIZE`) did not exist before this patch set showed up, `getauxval()` will return zero on older kernels. 

The first of the new values in `struct rseq` is called `node_id` and it contains exactly that: the ID number of the NUMA node on which the current thread is running. This is evidently useful for some memory allocators and, as noted in [the patch changelog](<https://lwn.net/ml/linux-kernel/20220218210633.23345-4-mathieu.desnoyers@efficios.com/>), supports (in conjunction with the already-present CPU ID) an entirely user-space implementation of [`getcpu()`](<https://man7.org/linux/man-pages/man2/getcpu.2.html>). 

The other new value is a bit further off the beaten path: it is called `vm_vcpu_id`. Like the `cpu_id` field in the same structure, it contains an integer ID number identifying the CPU on which the thread is running. But, while `cpu_id` contains the CPU's ID number as known by the kernel (and the rest of the system), `vm_vcpu_id` has no connection with the actual CPU number; it is a virtual number managed by the kernel in a process-private number space. 

This new CPU ID appears to be aimed at the needs of programs running threads on a relatively small number of CPUs in a large system. Remember that `rseq()` is aimed at helping programs access per-CPU data structures; such structures usually take the form of an array indexed by the current CPU ID number. That array must be large enough to hold an entry for every CPU in the system, and every entry must be properly initialized and maintained. 

That is just part of the task of working with per-CPU data structures. But imagine a smallish program, with a mere dozen threads or so, running on a large server with, say, 128 CPUs. Those threads may migrate over those CPUs as they run, or they may be bound to a specific subset of CPUs; either way, that per-CPU data structure must be set up for all 128 CPUs, which is not particularly efficient. It would be much nicer to match the "per-CPU" array size to the size of the program rather than that of the system it happens to be running on. 

That is the purpose of the virtual CPU ID number. These numbers are assigned by the kernel when a thread is scheduled onto a (real) CPU; the kernel takes pains to ensure that all concurrently running threads in the same process have different virtual CPU ID numbers. Those numbers are assigned from their own space, though, and are chosen to be close to zero. That leaves the program with fewer possible CPU numbers to deal with while preserving the benefits of working with per-CPU data structures. 

That does raise an interesting question, though: how does an application developer know what the range of possible virtual-CPU numbers is? When asked, Desnoyers [explained](<https://lwn.net/ml/linux-kernel/1323451367.108396.1645811762372.JavaMail.zimbra@efficios.com/>): 

> I would expect the user-space code to use some sensible upper bound as a hint about how many per-vcpu data structure elements to expect (and how many to pre-allocate), but have a "lazy initialization" fall-back in case the vcpu id goes up to the number of configured processors - 1\. 

One might expect the virtual-CPU ID to be bounded by the number of running threads, but the full story is more complicated than that. Using this feature, thus, will require a bit of additional complexity on the user-space side. 

Managing these virtual CPU IDs has a potential downside on the kernel side of the API as well: a certain amount of the work must be done in the scheduler's context-switch path, which is one of the hottest and most performance-critical paths in the kernel. Adding overhead there is not welcome. Desnoyers has duly taken a number of steps to minimize that overhead; they are described in [this patch changelog](<https://lwn.net/ml/linux-kernel/20220218210633.23345-10-mathieu.desnoyers@efficios.com/>). For example, a context switch between two threads of the same program just moves the virtual CPU ID from the outgoing thread to the incoming one, with no atomic operations required. Single-threaded programs are handled specially, and there is a special cache of virtual CPU IDs attached to each run queue which can be used to avoid atomic operations as well. 

Benchmarks included in that changelog show that the performance impact of these changes is small in most cases. Whether that will be enough to get the patches past the scheduler maintainers remains to be seen, though; they have yet to comment on this version of the series. Should this mechanism eventually be merged, though, it will be another tool available to developers looking for the best scalability possible in multithreaded applications.
