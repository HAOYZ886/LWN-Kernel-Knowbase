---
title: "Modernizing swapping: introducing the swap table"
url: https://lwn.net/Articles/1056405/
date: "February 2, 2026"
category: "Memory management-Swapping"
author: "By Jonathan Corbet February 2, 2026"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 2, 2026

The kernel's swap subsystem is a complex and often unloved beast. It is also a critical component in the memory-management subsystem and has a significant impact on the performance of the system as a whole. At the 2025 Linux Storage, Filesystem, Memory-Management and BPF Summit, Kairui Song [outlined a plan](<https://lwn.net/Articles/1016136/>) to simplify and optimize the kernel's swap code. A [first installment of that work](<https://lwn.net/ml/all/20250916160100.31545-1-ryncsn@gmail.com/>), written with help from Chris Li, was merged for the 6.18 release. This article will catch up with the 6.18 work, setting the stage for a future look at the changes that are yet to be merged. 

In a virtual-memory system, memory shortages must be addressed by reclaiming RAM and, if necessary, writing its contents to the appropriate persistent backing store. For file-backed memory, the file itself is that backing store. Anonymous memory — the memory that holds the variables and data structures used by a process — lacks that natural backing store, though. That is where the swap subsystem comes in: it provides a place to write anonymous pages when the memory they occupy is needed for other uses. Swapping allows unused (or seldom-used) pages to be pushed out to slower storage, making the system's RAM available for data that is currently in use. 

#### A quick swap-subsystem primer

A full description of the kernel's swap subsystem would be lengthy indeed; there is a lot of complexity, much of which has built up over time. What follows is a partial, simplified overview of how the swap subsystem looked in the 6.17 kernel, which can then be used as a base for understanding the subsequent changes. 

The swap subsystem uses one or more swap files, which can be either partitions on a storage device or ordinary files within a filesystem. Inside the kernel, active swap files are described by [`struct swap_info_struct`](<https://elixir.bootlin.com/linux/v6.17.13/source/include/linux/swap.h#L294>), but are usually referred to using a simple integer index instead. Each file is divided into page-sized slots; any given slot in the kernel's swap areas can be identified using the [`swp_entry_t`](<https://elixir.bootlin.com/linux/v6.17.13/source/include/linux/mm_types.h#L285>) type: 

```
typedef struct {
    	unsigned long val;
        } swp_entry_t;
```

This `long` value is divided into two fields: the upper six bits are the index number of the swap file (which, for extra clarity, is called the "type" in the swap code), and the rest is the slot number within the file. There is [a set of simple functions](<https://elixir.bootlin.com/linux/v6.17.13/source/include/linux/swapops.h#L83>) used to create swap entries and get the relevant information back out. 

Note that the above describes the architecture-independent form of the swap entry; each architecture will also have an architecture-dependent version that is used in page-table entries. Curious readers can look at [the x86_64 macros](<https://elixir.bootlin.com/linux/v6.18.6/source/arch/x86/include/asm/pgtable_64.h#L183>) that convert between the two formats. Within the swap subsystem itself, though, the architecture-independent version of the swap entry is used. 

An overly simplified description of swapping would be something like: when the memory-management subsystem decides to reclaim an anonymous page, it selects a swap slot, writes the page's contents into that slot, then stores the associated swap entry in the page-table entry (using the architecture-dependent format) with the "present" bit cleared. The next attempt to reference that page will result in a page fault; the kernel will see the swap entry, allocate a new page, read the contents from the swap file, then update the page-table entry accordingly. 

The truth of the matter is that things are rather more complex than that. For example, writing a page to the swap file takes time, and the page itself cannot be reclaimed until the write is complete. So, when the reclaim decision is made, the page is put into the swap cache, which is, in many ways, the analog of the page cache used for file-backed pages. Saying that a page is in the swap cache really only means that a swap entry has been assigned; the page itself may or may not still be resident in RAM. If a fault happens on that page while the writing process is underway, that page can be quickly reactivated, despite being in the swap cache. 

All of this means that the swap subsystem has to keep track of the status of every page in the swap cache, and that status involves more than just the swap slot that was assigned. To that end, in kernels prior to 6.18, the swap subsystem maintained an array called [`swapper_spaces`](<https://elixir.bootlin.com/linux/v6.17.13/source/mm/swap_state.c#L39>) that contained pointers to arrays of [`address_space`](<https://elixir.bootlin.com/linux/v6.17.13/source/include/linux/fs.h#L485>) structures. That structure is used to maintain the mapping between an address space (the bytes of a file, or the slots of a swap file) and the storage that backs up that space. It provides a set of operations that can be used to move pages between RAM and that backing store. Using `struct address_space` means, among other things, that much of the code that works with the page cache can also operate with the swap cache. 

Another reason to use `struct address_space` is the [XArray](<https://docs.kernel.org/core-api/xarray.html>) data structure associated with it. For a swap file, that data structure contains the current status of each slot in the file, which can be any of: 

  * The slot is empty. 
  * There is a page assigned to the slot, but that page is also resident in RAM; in that case, the XArray entry is a pointer to the page (more precisely, the folio containing the page) itself. 
  * There is a page assigned, but it exists only in the swap file. In that case, the entry contains "shadow" information used by the memory-management system to detect pages that are quickly faulted in after being swapped out. (See [this 2012 article](<https://lwn.net/Articles/495543/>) for an overview of this mechanism). 

For extra fun, there is not a single `address_space` structure and XArray for each swap file. Instead, the file is divided into 64MB chunks, and a separate `address_space` structure is created for each. This design helps to spread the management of swap entries across multiple XArrays, reducing contention and increasing scalability on larger systems where a lot of swapping is taking place. The `swapper_spaces` entry for a swap file, thus, points to an array of `address_space` structures; a 1GB swap file, for example, would be managed with an array of 16 of these structures. 

There is one more complication (for the purpose of this discussion — there are many others as well) in the management of swap slots. Each swap device is also divided into a set of swap clusters, represented by [`struct swap_cluster_info`](<https://elixir.bootlin.com/linux/v6.17.13/source/include/linux/swap.h#L238>); these clusters are usually 2MB in size. Swap clusters make the management of swap files more scalable; each CPU in the system maintains a cache of swap clusters that have been assigned to it. The associated swap entries can then be managed entirely locally to the CPU, with cross-CPU access only needed when clusters must be allocated or freed. Swap clusters reduce the amount of scanning of the global swap map needed to work with swap entries, but the appropriate XArray must still be used to obtain or modify the status of a given slot. 

#### The swap table

With that background in place, it is possible to look at the changes made for 6.18. They start with the understanding that the swap-subsystem code that deals with swap entries already has access to the swap clusters those entries belong to. Keeping the status information with the clusters would allow the elimination of the XArrays, which can be replaced with simple C arrays of swap entries. The smaller granularity of the swap clusters serves to further localize the management of swap entries, which should improve scalability. 

So the phase-1 patch set augments the `swap_cluster_info` structure; the [post-6.17 version of that structure](<https://elixir.bootlin.com/linux/v6.19-rc5/source/mm/swap.h#L21>) contains a new array pointer: 

```
atomic_long_t __rcu *table;
```

The new `table` array, which is designed to occupy exactly one page on most architectures, is allocated dynamically, reducing the swap subsystem's memory use when the swap files are not full. Each entry in the table is the same `swp_entry_t` value seen above, describing the status of one page in the swap cache. The swap code has been reworked to use this new organization, with many of the internal APIs needing minimal or no changes. The arrays of `address_space` structures covering 64MB each are gone; the XArrays are no longer needed, and the address-space operations can be provided by a single structure, called [`swap_space`](<https://elixir.bootlin.com/linux/v6.19-rc5/source/mm/swap.h#L201>). 

In summary, where the kernel previously divided swap areas using two independent clustering mechanisms (the `address_space` structures and the swap clusters), now it only has one clustering scheme that increases the locality of many swap operations. The end result, at this stage, is ""up to ~5-20% performance gain in throughput, RPS or build time for benchmark and workload tests"", according to Song. This speed improvement is entirely due to the removal of the XArray lookups and the reduction in contention that comes from managing swap space in smaller chunks. 

That is the state of affairs as of 6.18. As significant as this change is, it is only the beginning of the project to simplify and improve the kernel's swap code. The 6.19 kernel did not significantly advance this work, but there are two other installments under consideration, one of which is seemingly poised for the 7.0 release. Those changes will be covered in the [second part of this series](<https://lwn.net/Articles/1057102/>).
