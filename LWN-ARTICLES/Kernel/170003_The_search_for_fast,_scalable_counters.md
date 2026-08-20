---
title: "The search for fast, scalable counters"
url: https://lwn.net/Articles/170003/
date: "February 1, 2006"
category: "Per-CPU variables; atomic t; local t"
author: 
---

The kernel needs to count a lot of things. There are counters for networking statistics, usage of various resources, and so on. One would ordinarily think that operating a counter would be a relatively straightforward task, but ordinarily simple things can become complicated in the kernel context, especially when the number of processors involved gets large. 

In theory, a counter is just a simple integer variable. In an SMP environment, however, that variable must be protected against concurrent updates, or it will eventually get corrupted. The tool that kernel hackers reach for first in this situation is the `atomic_t` type. Atomic variables are simple integers with a set of atomic operations. If you have an `atomic_t` variable called `counter`, that counter can be incremented with a call like: 

```
atomic_inc(&counter);
```

and its value will be changed in an SMP-safe, interrupt-safe manner. These operations are relatively fast, being hand-coded to use the mechanisms provided by each host architecture. In many cases, an `atomic_t` counter is the best solution to the problem. 

The problem with `atomic_t` counters is that they use expensive locked operations, and they require that the current CPU obtain exclusive cache access for the variable. A frequently-modified atomic counter can cause a cache line to bounce constantly between CPUs, impacting the performance of the entire system. As an example, consider [this patch set](<http://lwn.net/Articles/169763/>) from Ravikiran Thirumalai. He replaced a single counter (the `memory_allocated` field of the `proto` structure) in the networking code with a more SMP-friendly counter, and reported a 5% improvement in an Apache benchmark on an eight-processor system. 5% is a nice improvement for changing a single counter, but it seems that perhaps even better results could be had. 

Ravikiran replaced the `atomic_t` counter with the `percpu_counter` type. These counters use per-CPU variables to hold a CPU-local count. Modifying that count is fast, since it is local to the given CPU, no locking is required, and no cache lines need be moved from other processors. If any given processor's count exceeds a given threshold, its value is added to a (spinlock-protected) global count, and the CPU-local count is set back to zero. Queries of the counter look only at the global count. The result is a counter which is somewhat approximate, but quite fast. In many cases, an "almost right" count is entirely good enough. 

Per-CPU counters become increasingly inaccurate as the number of processors grows, however. Each processor has a certain residual count which has not yet been folded into the global count. In situations where counters tend to increase, the result will be a global count which underestimates the real value, and which is increasingly wrong on larger systems. Per-CPU counters are also memory-intensive, partly due to inefficiencies in how per-CPU variables are allocated. 

So the discussion wandered toward [another possibility](<https://lwn.net/Articles/170050/>) implemented with the somewhat obscure `local_t` type. This type is apparently intended to function as a sort of `atomic_t` which is only visible to a single CPU; it is currently only used in two places in the kernel: to manage module reference counts and in the x86-64 architecture code. It supports a set of operations similar to `atomic_t`: `local_set()`, `local_read`, `local_add()`, etc. There is also a set of variants (cpu_local_set(), ...) intended for use with a `local_t` declared as a per-CPU variable. The default implementation uses `atomic_t` for 32-bit systems and a strange three-variable structure for 64-bit systems. All architectures are encouraged to reimplement the type in a more efficient, interrupt-safe manner, however, and that has been done for several of them. 

The `local_t` solution would set up two counters for each CPU, a flag saying which of the two is in use, and a global count. For many operations, they would behave just like `percpu_counter`, and they could yield the same approximate answer. Should a precise count be needed, however, the "which counter" bit would be flipped and all of the per-CPU offsets summed. The result would be an exact count at the time the bit was flipped, at the cost of taking a spinlock and iterating through the array. 

All of this starts to look a little elaborate, however, and that may be the point where kernel developers lose interest. A counter should only be so complex, and making the code more twisted can only improve things to a point. Sooner or later, people will decide that there are more important things to be working on.
