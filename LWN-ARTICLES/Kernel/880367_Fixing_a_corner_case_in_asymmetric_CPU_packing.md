---
title: Fixing a corner case in asymmetric CPU packing
url: https://lwn.net/Articles/880367/
date: "January 7, 2022"
category: Scheduler; big.LITTLE
author: "January 7, 2022 This article was contributed by Marta Rybczyńska"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

January 7, 2022

This article was contributed by Marta Rybczyńska

Linux supports processor architectures where CPUs in the same system might have different processing capacities; for example, the Arm [big.LITTLE](<https://en.wikipedia.org/wiki/ARM_big.LITTLE>) systems combine fast, power-hungry CPUs with slower, more efficient ones. Linux has also run for years on [simultaneous multithreading](<https://en.wikipedia.org/wiki/Simultaneous_multithreading>) (SMT) architectures, where one CPU executes multiple independent execution threads and is seen as if it were multiple cores. There are architectures that mix both approaches. A recent discussion on a [patch set](<https://lwn.net/ml/linux-kernel/20210911011819.12184-1-ricardo.neri-calderon@linux.intel.com/>) submitted by Ricardo Neri shows that, on these systems, the scheduler might distribute tasks in an inefficient way. 

#### Simultaneous multithreading

SMT functionality has been present in architectures like PowerPC and x86 for years. On an SMT system, a CPU can run instructions from two (or more) separate execution contexts. Each logical thread is visible as a separate CPU, so one physical CPU running two threads will be seen in Linux as two CPUs. SMT processors using the same hardware in this way are often called "siblings". Operating systems have little control over an SMT processor's decisions on how to divide its resources between execution contexts. 

SMT allows better use of a processor's resources because, when one execution path is stalled (waiting for memory, for example), the physical CPU can execute instructions from other threads. However, doubling the number of threads in a processor does not normally double its processing capacity. Both threads are sharing the same resources, and the SMT mode is most efficient when the system is under low load. Two SMT threads thus have a lower capacity than two physical CPU cores. 

This reduced capacity needs to be reflected in the scheduler, but the exact value of the reduction depends on the load, so the kernel needs to use a heuristic. Linux models this by reducing the CPU priority (which regulates how likely the CPU is to be chosen to run a given task) for the second (and following) CPU threads running on the same hardware.

Users can view sibling CPUs in the topology information available on their systems in `/sys/devices/system/cpu/cpu _X_ /topology/core_cpus_list` (where `_X_` is the number of a CPU on the system). For example, in a 12-core system with SMT: 

```
$cat /sys/devices/system/cpu/cpu0/topology/core_cpus_list
        0,6
```

The result means that user-visible cores 0 and 6 are SMT siblings, so they are using the same hardware core. 

#### `ASYM_PACKING`

Asymmetric packing ([`SD_ASYM_PACKING`](<https://elixir.bootlin.com/linux/v5.15.11/source/include/linux/sched/sd_flags.h#L140>) in the scheduler) is a feature [originally added](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v5.15-rc4&id=532cb4c401e225b084c14d6bd6a2f8ee561de2f1>) for the PowerPC architecture in 2010. It handles a case when the scheduler can obtain better processor performance by moving tasks to certain CPUs and leaving others idle. The busy CPUs can then move to a lower SMT mode (running fewer threads) and obtain higher overall system performance. [The SMT modes documentation](<https://www.ibm.com/docs/en/linux-on-systems?topic=linuxonibm/performance/tuneforsybase/smtsettings.htm>) for PowerPC includes some examples.

The `ASYM_PACKING` mode has witnessed a number of reworks and is currently supported on x86 and PowerPC. The support for x86 includes a way to support cores that might have a higher frequency than others, for example using the [Turbo Boost Max Technology (ITMT) 3.0](<https://lwn.net/Articles/702371/>) feature. A short [slide set](<https://blog.linuxplumbersconf.org/2016/ocw/system/presentations/3813/original/plumbers_fcore_v2.pdf>) from the 2016 Linux Plumbers Conference explains the work in a little more detail.

#### Mixing SMT and `ASYM_PACKING`

Neri observed some undesired scheduling behavior on a system with three distinct CPU priorities. This system contains high-performance cores (Intel Core) with their SMT siblings, along with lower-performance cores (Intel Atom). The efficient scheduling approach in this case (for some workloads at least) is to use the high-performance cores first (but without their SMT secondary threads), then the lower-performance cores, leaving the SMT secondary cores for last. The scheduler was, instead, putting tasks on the high-performance cores and their SMT siblings, leaving the other cores idle. As a result, tasks were contending for processor resources while independent CPUs remained idle.

To understand this problem, consider the example from [one of Neri's patches](<https://lwn.net/ml/linux-kernel/20210911011819.12184-2-ricardo.neri-calderon@linux.intel.com/>). Imagine a system with two physical CPUs with different priorities: 60 and 30 respectively. Both of them have SMT siblings. The kernel assigns SMT priorities using an [equation](<https://elixir.bootlin.com/linux/v5.15.11/source/arch/x86/kernel/itmt.c#L201>) in the x86-specific code: 

```
smt_prio = prio * smp_num_siblings / i;
```

where `smt_prio` is the effective priority, `prio` is the original priority of the CPU, `smp_num_siblings` is the number of siblings for each CPU (the value is two in Neri's case), and `i` is the sibling number assigned to the given physical CPU, starting from one. According to the formula, the resulting priorities are 120 for the main thread of the first CPU and 60 for its SMT sibling. For the second CPU, the main thread gets a priority of 60, and the sibling a priority of 30. In this case, the SMT sibling and the lower-performance main thread will have the same priorities.

Neri wanted to change the scheduler to assign tasks to the main thread of the second (physical) CPU before using the SMT sibling CPUs. To that end, he proposed a modification to the formula so that it would divide by _the square_ of the sibling number: 

```
smt_prio = prio * smp_num_siblings / (i*i);
```

In this case, the priorities will be 120 and 30 for the threads of the first CPU; then 60 and 15 for the threads of the second CPU. Tasks will thus be scheduled first on both main threads.

Neri's patch set makes another change when it comes to the scheduler's load-balancing decisions, and specifically when the scheduler decides to move a task from one CPU to another to even out the load on the system. When considering whether to move a task from a source CPU to a new target CPU, the scheduler considers whether the target has SMT siblings; if not, it can receive tasks from an SMT source CPU that has at least two busy threads. If only one sibling in the source CPU is busy, tasks will be moved only if the target CPU has a higher priority than the source. 

#### Summary

Scheduling performance improves by a few percent in some cases, according to the benchmark results presented in the cover letter, though it is smaller in most cases. Benchmarks also show also some cases with performance degradation, but Neri gave no explanation for this result.

The change has been merged for the 5.16 kernel, so owners of such systems should see a change in scheduling and, hopefully, better performance. This fix covers one scheduling corner case; there is no reason to think it was the only one. We should expect to see more adjustments in scheduling on asymmetric CPUs in the future.
