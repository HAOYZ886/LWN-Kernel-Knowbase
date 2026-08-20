---
title: "Better per-CPU variables"
url: https://lwn.net/Articles/258238/
date: "November 12, 2007"
category: "Memory management-Internal API; Per-CPU variables"
author: "By Jonathan Corbet November 12, 2007"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 12, 2007

One of the great advantages of multiprocessor computers is the fact that main memory is available to all processors on the system. This ability to share data gives programmers a great deal of flexibility. One of the first things those programmers learn (or should learn), however, is that actually sharing data between processors is to be avoided whenever possible. The sharing of data - especially data which changes - causes all kinds of bad cache behavior and greatly reduced performance. The recently-concluded [What every programmer should know about memory](<http://lwn.net/Articles/250967/>) series covers these problems in great detail. 

Over the years, kernel developers have made increasing use of per-CPU data in an effort to minimize memory contention and its associated performance penalties. As a simple example, consider the disk operation statistics maintained by the block layer. Incrementing a global counter for every disk operation would cause the associated cache line to bounce continually between processors; disk operations are frequent enough that the performance cost would be measurable. So each CPU maintains its own set of counters locally; it never has to contend with any other CPU to increment one of those counters. When a total count is needed, all of the per-CPU counters are added up. Given that the counters are queried far more rarely than they are modified, storing them in per-CPU form yields a significant performance improvement. 

In current kernels, most of these per-CPU variables are managed with an array of pointers. So, for example, the `kmem_cache` structure (as implemented by the SLUB allocator) contains this field: 

```
struct kmem_cache_cpu *cpu_slab[NR_CPUS];
```

![\[percpu array\]](https://static.lwn.net/images/ns/kernel/percpu-array.png) Note that the array is dimensioned to hold one pointer for every possible CPU in the system. Most deployed computers have fewer than the maximum number of processors, though, so there is, in general, no point in allocating `NR_CPUS` objects for that array. Instead, only the entries in the array which correspond to existing processors are populated; for each of those processors, the requisite object is allocated using `kmalloc()` and stored into the array. The end result is an array that looks something like the diagram on the right. In this case, per-CPU objects have been allocated for four processors, with the remaining entries in the array being unallocated. 

A quick look at the diagram immediately shows one potential problem with this scheme: each of these per-CPU arrays is likely to have some wasted space at the end. `NR_CPUS` is a configuration-time constant; most general-purpose kernels (e.g. those shipped by distributors) tend to have `NR_CPUS` set high enough to work on most or all systems which might reasonably be encountered. In short, `NR_CPUS` is likely to be quite a bit larger than the number of processors actually present, with the result that there will be a significant amount of wasted space at the end of each per-CPU array. 

In fact, Christoph Lameter noticed that are more problems than that; in response, he has posted [a patch series for a new per-CPU allocator](<https://lwn.net/Articles/258248/>). The deficiencies addressed by Christoph's patch (beyond the wasted space in each per-CPU array) include: 

  * If one of these per-CPU arrays is embedded within a larger data structure, it may separate the other variables in that structure, causing them to occupy more cache lines than they otherwise would. 

  * Each CPU uses exactly one pointer from that array (most of the time); that pointer will reside in the processor's data cache while it is being used. Cache lines hold quite a bit more than one pointer, though; in this case, the rest of the cache line is almost certain to hold the pointers for the other CPUs. Thus, scarce cache space is being wasted on completely useless data. 

  * Accessing the object requires two pointer lookups - one to get the object pointer from the array, and one to get to the object itself. 

Christoph's solution is quite simple in concept: turn all of those little per-CPU arrays into one big per-CPU array. With this scheme, each processor is allocated a dedicated range of memory at system initialization time. These ranges are all contiguous in the kernel's virtual address ![\[New percpu structure\]](https://static.lwn.net/images/ns/kernel/percpu-noarray.png) space, so, given a pointer to the per-CPU area for CPU 0, the area for any other processor is just a pointer addition away. 

When a per-CPU object is allocated, each CPU gets a copy obtained from its own per-CPU area. Crucially, the offset into each CPU's area is the same, so the address of any CPU's object is trivially calculated from the address of the first object. So the array of pointers can go away, replaced by a single pointer to the object in the area reserved for CPU 0\. The resulting organization looks (with the application of sufficient imagination) something like the diagram to the right. For a given object, there is only a single pointer; all of the other versions of that object are found by applying a constant offset to that pointer. 

The interface for the new allocator is relatively straightforward. A new per-CPU variable is created with: 

```
#include <linux/cpu_alloc.h>
    
        void *per_cpu_var = CPU_ALLOC(type, gfp_flags);
```

This call will allocate a set of per-CPU variables of the given `type`, using the usual `gfp_flags` to control how the allocation is performed. A pointer to a specific CPU's version of the variable can be had with: 

```
void *CPU_PTR(per_cpu_var, unsigned int cpu);
        void *THIS_CPU(per_cpu_var);
```

The `THIS_CPU()` form, as might be expected, returns a pointer to the version of the variable allocated for the current CPU. There is a `CPU_FREE()` macro for returning a per-CPU object to the system. Christoph's patch converts all users of the existing per-CPU interface and ends by removing that API altogether. 

There are a number of advantages to this approach. There's one less pointer operation for each access to a per-CPU variable. The same pointer is used on all processors, resulting in smaller data structures and better cache line utilization. Per-CPU variables for a given processor are grouped together in memory, which, again, should lead to better cache use. All of the memory wasted in the old pointer arrays has been reclaimed. Christoph also claims that this mechanism, by making it easier to keep track of per-CPU memory, makes the support of CPU hotplugging easier. 

The amount of discussion inspired by this patch set has been relatively low. There were complaints about the UPPER CASE NAMES used by the macros. The biggest complaint, though, has to do with the way the static per-CPU areas bloat the kernel's data space. On some architectures it makes the kernel too large to boot, and it's a real cost on all architectures. Just how this issue will be resolved is not yet clear. If a solution can be found, the new per-CPU code has a good chance of getting into the mainline when the 2.6.25 merge window opens.
