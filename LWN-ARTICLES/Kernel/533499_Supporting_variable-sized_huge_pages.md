---
title: "Supporting variable-sized huge pages"
url: https://lwn.net/Articles/533499/
date: "January 23, 2013"
category: "Huge pages; Memory management-Huge pages"
author: "By Michael Kerrisk January 23, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Michael Kerrisk**  
January 23, 2013

[Huge pages](<https://lwn.net/Articles/374424/>) are an optimization technique designed to increase virtual memory performance. The idea is that instead of a traditional small virtual memory page size (4 kB on most architectures), an application can employ (much) larger pages (e.g., 2 MB or 1 GB on x86-64). For applications that can make full use of larger pages, huge pages provide a number of performance benefits. First, a single page fault can fault in a large block of memory. Second, larger page sizes equate to shallower page tables, since fewer page-table levels are required to span the same range of virtual addresses; consequently, less time is required to traverse page table entries when translating virtual addresses to physical addresses. Finally, and most significantly, since entries for huge pages in the translation lookaside buffer (TLB) span much greater address ranges, there is an increased chance that a virtual address already has a match in one of the limited set of entries currently cached in the TLB, thus obviating the need to traverse page tables. 

Applications can explicitly request the use of huge pages when making allocations, using either `shmget()` with the `SHM_HUGETLB` flag (since Linux 2.6.0) or `mmap()` with the `MAP_HUGETLB` flag (since Linux 2.6.32). It's worth noting that explicit application requests are not needed to employ huge pages: the [transparent huge pages](<https://lwn.net/Articles/423584/>) feature merged in Linux 2.6.38 allows applications to gain much of the performance benefit of huge pages without making any changes to application code. There is, however, a limitation to these APIs: they provide no way to specify the size of the huge pages to be used for an allocation. Instead, the kernel employs the "default" huge page size. 

Some architectures only permit one huge page size; on those architectures, the default is in fact the only choice. However, some modern architectures permit multiple huge page sizes, and where the system administrator has configured the system to provide huge page pools of different sizes, applications may want to choose the page size used for their allocation. For example, this may be useful in a NUMA environment, where a smaller huge page size may be suitable for mappings that are shared across CPUs, while a larger page size is used for mappings local to a single CPU. 

A [patch](<https://lwn.net/Articles/533650/>) by Andi Kleen that was accepted during the 3.8 merge window extends the `shmget()` and `mmap()` system calls to allow the caller to select the size used for huge page allocations. These system calls have the following prototypes: 

```
void *mmap(void *addr, size_t length, int prot, int flags,
                   int fd, off_t offset);
        int shmget(key_t key, size_t size, int shmflg);
```

Neither of those calls provides an argument that can be directly used to specify the desired page size. Therefore, Andi's patch shoehorns the value into some bits that are currently unused in one of the arguments of each call--in the `flags` argument for `mmap()` and in the `shmflg` argument for `shmget()`. 

In both system calls, the huge page size is encoded in the six bits from 26 through to 31 (i.e., the bit mask `0xfc000000`). The value in those six bits is the base-two log of the desired page size. As a special case, if the value encoded in the bits is zero, then the kernel selects the default huge page size. This provides binary backward compatibility for the interfaces. If the specified page size is not supported by the architecture, then `shmget()` and `mmap()` fail with the error `ENOMEM`. 

An application can manually perform the required base-two log calculation and bit shift to generate the required bit-mask value, but this is clumsy. Instead, an architecture can define suitable constants for the huge page sizes that it supports. Andi's patch defines two such constants corresponding to the available page sizes on x86-64: 

```
#define SHM_HUGE_SHIFT  26
        #define SHM_HUGE_MASK   0x3f
        /* Flags are encoded in bits (SHM_HUGE_MASK << SHM_HUGE_SHIFT) */
    
        #define SHM_HUGE_2MB    (21 << SHM_HUGE_SHIFT)   /* 2 MB huge pages */
        #define SHM_HUGE_1GB    (30 << SHM_HUGE_SHIFT)   /* 1 GB huge pages */
```

Corresponding `MAP_*` constants are defined for use in the `mmap()` system call. 

Thus, to employ a 2 MB huge page size when calling `shmget()`, one would write: 

```
shmget(key, size, flags | SHM_HUGETLB | SHM_HUGE_2MB);
```

That is, of course, the same as this manually calculated version: 

```
shmget(key, size, flags | SHM_HUGETLB | (21 << HUGE_PAGE_SHIFT));
```

In passing, it's worth noting that an application can determine the default page size by looking at the `Hugepagesize` entry in `/proc/meminfo` and can, if the kernel was configured with `CONFIG_HUGETLBFS`, discover the available page sizes on the system by scanning the directory entries under `/sys/kernel/mm/hugepages`. 

One concern [raised](<https://lwn.net/Articles/533681/>) by your editor when reviewing an earlier version of Andi's patch was whether the bit space in the `mmap()` `flags` argument is becoming exhausted. Exactly how many bits are still unused in that argument turns out to be a little difficult to determine, because different architectures define the same flags with different values. For example, the `MAP_HUGETLB` flag has the values 0x4000, 0x40000, 0x80000, or 0x100000, depending on the architecture. It turns out that before Andi's patch was applied, there were only around 11 bits in `flags` that were unused across all architectures; now that the patch has been applied, just six are left. 

The day when the `mmap()` `flags` bit space is exhausted seems to be slowly but steadily approaching. When that happens, either a new `mmap()`-style API with a 64-bit `flags` argument will be required, or, as Andi [suggested](<https://lwn.net/Articles/533682/>), unused bits in the `prot` argument could be used; the latter option would be easier to implement, but would also further muddy the interface of an already complex system call. In any case, concerns about the API design didn't stop Andrew Morton from accepting the patch, although he was prompted to [remark](<https://lwn.net/Articles/533683/>) ""I can't say the userspace interface is a thing of beauty, but I guess we'll live."" 

The new API features will roll out in few weeks' time with the 3.8 release. At that point, application writers will be able to select different huge page sizes for different memory allocations. However, it will take a little longer before the `MAP_*` and `SHM_*` page size constants percolate through to the GNU C library. In the meantime, programmers who are in a hurry will have to define their own versions of these constants.
