---
title: Huge pages part 1 (Introduction)
url: https://lwn.net/Articles/374424/
date: "February 16, 2010"
category: "Huge pages; Memory management-Huge pages; hugetlbfs"
author: "February 16, 2010 This article was contributed by Mel Gorman"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

February 16, 2010

This article was contributed by Mel Gorman

[_Editor's note: this article is the first in a five-part series on the use of huge pages with Linux. We are most fortunate to have core VM hacker Mel Gorman as the author of these articles! The remaining installments will appear in future LWN Weekly Editions._] 

One of the driving forces behind the development of Virtual Memory (VM) was to reduce the programming burden associated with fitting programs into limited memory. A fundamental property of VM is that the CPU references a virtual address that is translated via a combination of software and hardware to a physical address. This allows information only to be paged into memory on demand (demand paging) improving memory utilisation, allows modules to be arbitrary placed in memory for linking at run-time and provides a mechanism for the protection and controlled sharing of data between processes. Use of virtual memory is so pervasive that it has been described as "one of the engineering triumphs of the computer age" [denning96], but this indirection is not without cost. 

Typically, the total number of translations required by a program during its lifetime will require that the page tables are stored in main memory. Due to translation, a virtual memory reference necessitates multiple accesses to physical memory, multiplying the cost of an ordinary memory reference by a factor depending on the page table format. To cut the costs associated with translation, VM implementations take advantage of the principal of locality [denning71] by storing recent translations in a cache called the Translation Lookaside Buffer (TLB) [casep78,smith82,henessny90]. The amount of memory that can be translated by this cache is referred to as the "TLB reach" and depends on the size of the page and the number of TLB entries. Inevitably, a percentage of a program's execution time is spent accessing the TLB and servicing TLB misses. 

The amount of time spent translating addresses depends on the workload as the access pattern determines if the TLB reach is sufficient to store all translations needed by the application. On a miss, the exact cost depends on whether the information necessary to translate the address is in the CPU cache or not. To work out the amount of time spent servicing the TLB misses, there are some simple formulas: 

> Cyclestlbhit = TLBHitRate * TLBHitPenalty 
> 
> Cyclestlbmiss_cache = TLBMissRatecache * TLBMissPenaltycache
> 
> Cyclestlbmiss_full = TLBMissRatefull * TLBMissPenaltyfull
> 
> TLBMissCycles = Cyclestlbmiss_cache \+ Cycles_tlbmiss_full
> 
> TLBMissTime = (TLB Miss Cycles)/(Clock rate) 

If the TLB miss time is a large percentage of overall program execution, then the time should be invested to reduce the miss rate and achieve better performance. One means of achieving this is to translate addresses in larger units than the base page size, as supported by many modern processors. 

Using more than one page size was identified in the 1990s as one means of reducing the time spent servicing TLB misses by increasing TLB reach. The benefits of huge pages are twofold. The obvious performance gain is from fewer translations requiring fewer cycles. A less obvious benefit is that address translation information is typically stored in the L2 cache. With huge pages, more cache space is available for application data, which means that fewer cycles are spent accessing main memory. Broadly speaking, database workloads will gain about 2-7% performance using huge pages whereas scientific workloads can range between 1% and 45%. 

Huge pages are not a universal gain, so transparent support for huge pages is limited in mainstream operating systems. On some TLB implementations, there may be different numbers of entries for small and huge pages. If the CPU supports a smaller number of TLB entries for huge pages, it is possible that huge pages will be slower if the workload reference pattern is very sparse and making a small number of references per-huge-page. There may also be architectural limitations on where in the virtual address space huge pages can be used. 

Many modern operating systems, including Linux, support huge pages in a more explicit fashion, although this does not necessarily mandate application change. Linux has had support for huge pages since around 2003 where it was mainly used for large shared memory segments in database servers such as Oracle and DB2. Early support required application modification, which was considered by some to be a major problem. To compound the difficulties, tuning a Linux system to use huge pages was perceived to be a difficult task. There have been significant improvements made over the years to huge page support in Linux and as this article will show, using huge pages today can be a relatively painless exercise that involves no source modification. 

This first article begins by installing some huge-page-related utilities and support libraries that make tuning and using huge pages a relatively painless exercise. It then covers the basics of how huge pages behave under Linux and some details of concern on NUMA. The second article covers the different interfaces to huge pages that exist in Linux. In the third article, the different considerations to make when tuning the system are examined as well as how to monitor huge-page-related activities in the system. The fourth article shows how easily benchmarks for different types of application can use huge pages without source modification. For the very curious, some in-depth details on TLBs and measuring the cost within an application are discussed before concluding. 

### 1 Huge Page Utilities and Support Libraries

There are a number of support utilities and a library packaged collectively as [libhugetlbfs](<http://sourceforge.net/projects/libhugetlbfs/>). Distributions may have packages, but this article assumes that libhugetlbfs 2.7 is installed. The latest version can always be cloned from git using the following instructions 

```
$ git clone git://libhugetlbfs.git.sourceforge.net/gitroot/libhugetlbfs/libhugetlbfs
      $ cd libhugetlbfs
      $ git checkout -b next origin/next
      $ make PREFIX=/usr/local
```

There is an `install` target that installs the library and all support utilities but there are `install-bin`, `install-stat` and `install-man` targets available in the event the existing library should be preserved during installation. 

The library provides support for automatically backing text, data, heap and shared memory segments with huge pages. In addition, this package also provides a programming API and manual pages. The behaviour of the library is controlled by environment variables (as described in the `libhugetlbfs.7` manual page) with a launcher utility `hugectl` that knows how to configure almost all of the variables. `hugeadm`, `hugeedit` and `pagesize` provide information about the system and provide support to system administration. `tlbmiss_cost.sh` automatically calculates the average cost of a TLB miss. `cpupcstat` and `oprofile_start.sh` provide help with monitoring the current behaviour of the system. Manual pages are available describing in further detail each utility. 

### 2 Huge Page Fault Behaviour

In the following articles, there will be discussions on how different type of memory regions can be created and backed with huge pages. One important common point between them all is how huge pages are faulted and when the huge pages are allocated. Further, there are important differences between shared and private mappings depending on the exact kernel version used. 

In the initial support for huge pages on Linux, huge pages were faulted at the same time as `mmap()` was called. This guaranteed that all references would succeed for shared mappings once `mmap()` returned successfully. Private mappings were safe until `fork()` was called. Once called, it's important that the child call `exec()` as soon as possible or that the huge page mappings were marked `MADV_DONTFORK` with `madvise()` in advance. Otherwise, a Copy-On-Write (COW) fault could result in application failure by either parent or child in the event of allocation failure. 

Pre-faulting pages drastically increases the cost of `mmap()` and can perform sub-optimally on NUMA. Since 2.6.18, huge pages were faulted the same as normal mappings when the page was first referenced. To guarantee that faults would succeed, huge pages were reserved at the time the shared mapping is created but private mappings do not make any reservations. This is unfortunate as it means an application can fail without `fork()` being called. `libhugetlbfs` handles the private mapping problem on old kernels by using `readv()` to make sure the mapping is safe to access, but this approach is less than ideal. 

Since 2.6.29, reservations are made for both shared and private mappings. Shared mappings are guaranteed to successfully fault regardless of what process accesses the mapping. 

For private mappings, the number of child processes is indeterminable so only the process that creates the mapping `mmap()` is guaranteed to successfully fault. When that process `fork()`s, two processes are now accessing the same pages. If the child performs COW, an attempt will be made to allocate a new page. If it succeeds, the fault successfully completes. If the fault fails, the child gets terminated with a message logged to the kernel log noting that there were insufficient huge pages. If it is the parent process that performs COW, an attempt will also be made to allocate a huge page. In the event that allocation fails, the child's pages are unmapped and the event recorded. The parent successfully completes the fault but if the child accesses the unmapped page, it will be terminated. 

### 3 Huge Pages and Swap

There is no support for the paging of huge pages to backing storage. 

### 4 Huge Pages and NUMA

On NUMA, memory can be local or remote to the CPU, with significant penalty incurred for remote access. By default, Linux uses a node-local policy for the allocation of memory at page fault time. This policy applies to both base pages and huge pages. This leads to an important consideration while implementing a parallel workload. 

The thread processing some data should be the same thread that caused the original page fault for that data. A general anti-pattern on NUMA is when a parent thread sets up and initialises all the workload's memory areas and then creates threads to process the data. On a NUMA system this can result in some of the worker threads being on CPUs remote with respect to the memory they will access. While this applies to all NUMA systems regardless of page size, the effect can be pronounced on systems where the split between worker threads is in the middle of a huge page incurring more remote accesses than might have otherwise occurred. 

This scenario may occur for example when using huge pages with OpenMP, because OpenMP does not necessarily divide its data on page boundaries. This could lead to problems when using base pages, but the problem is more likely with huge pages because a single huge page will cover more data than a base page, thus making it more likely any given huge page covers data to be processed by different threads. Consider the following scenario. A first thread to touch a page will fault the full page's data into memory local to the CPU on which the thread is running. When the data is not split on huge-page-aligned boundaries, such a thread will fault its data and perhaps also some data that is to be processed by another thread, because the two threads' data are within the range of the same huge page. The second thread will fault the rest of its data into local memory, but will still have part of its data accesses be remote. This problem manifests as large standard deviations in performance when doing multiple runs of the same workload with the same input data. Profiling in such a case may show there are more cross-node accesses with huge pages than with base pages. In extreme circumstances, the performance with huge pages may even be slower than with base pages. For this reason it is important to consider on what boundary data is split when using huge pages on NUMA systems. 

One work around for this instance of the general problem is to use MPI in combination with OpenMP. The use of MPI allows division of the workload with one MPI process per NUMA node. Each MPI process is bound to the list of CPUs local to a node. Parallelisation within the node is achieved using OpenMP, thus alleviating the issue of remote access. 

### 5 Summary

In this article, the background to huge pages were introduced, what the performance benefits can be and some basics of how huge pages behave on Linux. The next article (to appear in the near future) discusses the interfaces used to access huge pages. 

Read the successive installments: 

  * [Part 2](<http://lwn.net/Articles/375096/>): Interfaces 
  * [Part 3](<http://lwn.net/Articles/376606/>): Administration 
  * [Part 4](<https://lwn.net/Articles/378641/>): Benchmarking 
  * [Part 5](<http://lwn.net/Articles/379748/>): A deeper look at TLBs and costs 

Details of publications referenced in these articles can be found in the [bibliography](<https://lwn.net/Articles/379748/#bibliography>) at the end of Part 5. 

_This material is based upon work supported by the Defense Advanced Research Projects Agency under its Agreement No. HR0011-07-9-0002. Any opinions, findings and conclusions or recommendations expressed in this material are those of the author and do not necessarily reflect the views of the Defense Advanced Research Projects Agency._
