---
title: "Huge pages part 4: benchmarking with huge pages"
url: https://lwn.net/Articles/378641/
date: "March 17, 2010"
category: "Huge pages; Memory management-Huge pages"
author: "March 17, 2010 This article was contributed by Mel Gorman"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

March 17, 2010

This article was contributed by Mel Gorman

[_Editor's note: this is part 4 of Mel Gorman's series on support for huge pages in Linux. Parts[1](<http://lwn.net/Articles/374424/>), [2](<http://lwn.net/Articles/375096/>), and [3](<http://lwn.net/Articles/376606/>) are available for those who have not read them yet._] 

In this installment, a small number of benchmarks are configured to use huge pages - STREAM, sysbench, SpecCPU 2006 and SpecJVM. In doing so, we show that utilising huge pages is a lot easier than in the past. In all cases, there is a heavy reliance on the `hugeadm` to simplify the machine configuration and `hugectl` to configure `libhugetlbfs`. 

STREAM is a memory-intensive benchmark and, while its reference pattern has poor spacial and temporal locality, it can benefit from reduced TLB references. Sysbench is a simple OnLine Transaction Processing (OLTP) benchmark that can use Oracle, MySQL, or PostgreSQL as database backends. While there are better OLTP benchmarks out there, Sysbench is very simple to set up and reasonable for illustration. SpecCPU 2006 is a computational benchmark of interest to high-performance computing (HPC) and SpecJVM benchmarks basic classes of Java applications. 

### 1 Machine Configuration

The machine used for this study is a Terrasoft Powerstation described in the table below. 

> **Architecture** |  PPC64   
> ---|---  
> **CPU** |  PPC970MP with altivec   
> **CPU Frequency** |  2.5GHz   
> **# Physical CPUs** |  2 (4 cores)   
> **L1 Cache per core** |  32K Data, 64K Instruction   
> **L2 Cache per core** |  1024K Unified   
> **L3 Cache per socket** |  N/a   
> **Main Memory** |  8 GB   
> **Mainboard** |  Machine model specific   
> **Superpage Size** |  16MB   
> **Machine Model** |  Terrasoft Powerstation   
  
Configuring the system for use with huge pages was a simple matter of performing the following commands. 

```
$ hugeadm --create-global-mounts
        $ hugeadm --pool-pages-max DEFAULT:8G 
        $ hugeadm --set-recommended-min_free_kbytes
        $ hugeadm --set-recommended-shmmax
        $ hugeadm --pool-pages-min DEFAULT:2048MB
        $ hugeadm --pool-pages-max DEFAULT:8192MB
```

### 2 STREAM

[STREAM](<http://www.cs.virginia.edu/stream/>) [mccalpin07] is a synthetic memory bandwidth benchmark that measures the performance of four long vector operations: Copy, Scale, Add, and Triad. It can be used to calculate the number of floating point operations that can be performed during the time for the "average" memory access. Simplistically, more bandwidth is better. 

The C version of the benchmark was selected and used three statically allocated arrays for calculations. Modified versions of the benchmark using `malloc()` and `get_hugepage_region()` were found to have similar performance characteristics. 

The benchmark has two parameters: `N`, the size of the array, and `OFFSET`, the number of elements padding the end of the array. A range of values for `N` were used to generate workloads between 128K and 3GB in size. For each size of `N` chosen, the benchmark was run 10 times and an average taken. The benchmark is sensitive to cache placement and optimal layout varies between architectures; where the standard deviation of 10 iterations exceeded 5% of the throughput, `OFFSET` was increased to add one cache-line of padding between the arrays and the benchmark for that value of `N` was rerun. High standard deviations were only observed when the total working set was around the size of the L1, L2 or all caches combined. 

The benchmark avoids data re-use, be it in registers or in the cache. Hence, benefits from huge pages would be due to fewer faults, a slight reduction in TLB misses as fewer TLB entries are needed for the working set and an increase in available cache as less translation information needs to be stored. 

To use huge pages, the benchmark was first compiled with the libhugetlbfs `ld` wrapper to align the text and data sections to a huge page boundary [libhtlb09] such as in the following example. 

```
$ gcc -DN=1864135 -DOFFSET=0 -O2 -m64                     \
            -B /usr/share/libhugetlbfs -Wl,--hugetlbfs-align     \
            -Wl,--library-path=/usr/lib                          \
            -Wl,--library-path=/usr/lib64                        \
            -lhugetlbfs stream.c                                 \
            -o stream
    
       # Test launch of benchmark
       $ hugectl --text --data --no-preload ./stream
```

[![\[STREAM
benchmark result\]](https://static.lwn.net/images/ns/kernel/hugepage/stream-sm.png)](<http://lwn.net/Articles/378646/>) [This page](<http://lwn.net/Articles/378646/>) contains plots showing the performance results for a range of sizes running on the test machine; one of them appears to the right. Performance improvements range from 11.6% to 16.59% depending on the operation in use. Performance improvements would be typically lower for an X86 or X86-64 machine, likely in the 0% to 4% range.
