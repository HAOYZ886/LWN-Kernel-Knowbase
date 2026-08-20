---
title: "Modernizing swapping: the end of the swap map"
url: https://lwn.net/Articles/1057102/
date: "February 5, 2026"
category: "Memory management-Swapping"
author: "By Jonathan Corbet February 5, 2026"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 5, 2026

[The first installment in this series](<https://lwn.net/Articles/1056405/>) introduced several data structures in the kernel's swap subsystem and described work to replace some of those with a new "swap table" structure. The work did not stop there, though; there is more modernization of the swap subsystem queued for an upcoming development cycle, and even more for multiple kernel releases after that. Once that work is done, the swap subsystem will be both simpler and faster than it is now. 

The data structures introduced thus far include the swap cluster, which represents a 2MB set of swap slots within a swap file, and the new swap table, stored within the swap cluster, that tracks the state of each swap slot. The introduction of the swap table allowed the removal of entire arrays of [XArray](<https://docs.kernel.org/core-api/xarray.html>) structures that were, prior to the 6.18 kernel release, used to track the status of individual swap slots within a swap file. That was not a complete list of swap-related data structures, though. The first article, as a way of minimizing the complexity of the picture as much as possible, skipped over an important swap-subsystem component: the swap map. 

#### The swap map

The time has come to fill in that gap, as the swap map is the core target of the ongoing swap-improvement effort. At first glance, the swap map, as found in current kernels, is as simple as data structures get. There is one for each swap device, stored in [`struct swap_info_struct`](<https://elixir.bootlin.com/linux/v6.19-rc5/source/include/linux/swap.h#L260>), and declared as: 

```
unsigned char *swap_map;  /* vmalloc'ed array of usage counts */
```

This field points to an array with one byte for every slot in the swap device; the value stored in each byte is the number of references that exist to that swap slot. There will be one reference for every page-table entry pointing to that slot, regardless of whether the page assigned to that slot is resident in RAM. 

Of course, this is the swap code that is being discussed, so there are complications, and that some of the bits in the swap-map entries have special meaning. The most significant of those, for the purposes of this article, is bit six (`0x40`) of the reference count; it is called `SWAP_HAS_CACHE`, and it is used to indicate that a swap slot has a page assigned to it. There can be various windows of time where a swap slot is assigned, but no page-table references to that slot yet exist, leading to a reference count of zero. The `SWAP_HAS_CACHE` bit distinguishes that state from a slot being unassigned. 

This flag is also used as a sort of bit lock; there are numerous race conditions that might cause the kernel to try to swap in a page (or make other changes) multiple times in parallel. In such cases, the thread that succeeds in setting the `SWAP_HAS_CACHE` bit in the entry is the one that proceeds to do the work. This use of `SWAP_HAS_CACHE` as a synchronization mechanism has led to a number of problems over the years; the swap code has a number of delay-and-retry loops ([example](<https://elixir.bootlin.com/linux/v6.19-rc5/source/mm/swap_state.c#L465>)) waiting for this bit to clear. 

There are some other special values in the swap map; a value of `0x3f` (`SWAP_MAP_BAD`) means, for example, that the underlying storage is bad and should not be used. As a result, the maximum reference count that can be stored in the swap map (`SWAP_MAP_MAX` — `0x3e`) is 62\. That presents a problem; in cases where a large number of tasks are sharing an anonymous page, the number of references could easily exceed that value. The way this situation is handled is, to put it mildly, interesting. 

Every time that the reference count for a swap slot is incremented, a check must be made for overflow. Should the count already be at the maximum, the topmost bit (`0x80` — `COUNT_CONTINUED`) will be set, the count in the swap map will be set to zero, and a new page will be allocated to provide eight more-significant bits for the reference count (and for all the others on the same original swap-map page). That page will be linked to the swap-map page using the LRU list head in the associated [`page` structures](<https://elixir.bootlin.com/linux/v6.19-rc5/source/include/linux/mm_types.h#L42>). If an entry has a lot of references and the count in the overflow page also overflows, yet another page will be allocated and added to the list. 

The overflow pages only need to be accessed when the principal swap-map entry overflows or underflows, which is good considering that these operations are supposed to be fast. While the motivation behind this somewhat baroque design isn't documented anywhere, one can assume that, while the overflow case must be handled correctly, it is also relatively rare. Massive sharing of anonymous pages is not the common case. When reference counts are lower, this structure offers quick access and minimal memory overhead. 

#### Swap-cache bypass and `SWAP_HAS_CACHE`

One of the purposes of the swap cache is to hold (and track) folios that are under I/O to or from the swap device. If, for example, a page fault occurs on a swapped-out folio, a new folio will have to be allocated and its contents read from the swap file. That read operation can take some time, though. So the folio is added to the swap cache, the read operation is initiated, and the faulting process made to wait until the read is complete. Often, the swap subsystem will also attempt to read ahead of the current fault location, making a bet that the process will soon fault in subsequent pages as well. 

The situation changed a bit in the 2018 4.15 release, though. Once upon a time, swapping was mostly done to rotating storage devices, which are slow. Increasingly, though, swapping looks a lot like just copying data from one part of memory to another. The "swap device" may be a bank of slower memory, or it may be an in-memory compression scheme like [zram](<https://docs.kernel.org/admin-guide/blockdev/zram.html>). On such devices, swap I/O is no longer slow, and behavior like readahead may harm performance rather than helping it. 

In 4.15, Minchan Kim added the "swap bypass" feature. Specifically, if a swap device has the `SWP_SYNCHRONOUS_IO` flag (indicating that the device is so fast that I/O should be done synchronously) set, and if a specific slot in the swap map has a reference count of one, then a request to swap in the page stored in that slot will happen synchronously, readahead will not be performed, and the newly read page will not be added to the swap cache. This optimization added a fair amount of complexity to the swap subsystem, resulting in various bugs over time, but it also resulted in significantly better performance for swap-heavy workloads. That improvement was due to two factors: avoiding the relatively expensive swap-cache maintenance and preventing the use of readahead. 

Fast-forwarding now to 2026, first part of the [phase-two patch series](<https://lwn.net/ml/all/20251220-swap-table-p2-v5-0-8862a265a033@tencent.com>) from Kairui Song is dedicated to removing the bypass feature. The work done in the first phase — specifically the introduction of the swap table — made swap-cache operations much faster, to the point that there is no real value to bypassing the swap-cache even when fast swap devices are in use. Additional work in this series separates out the control of readahead and essentially disables its use entirely for fast devices. Having all swap I/O go through the swap cache simplifies the code and reduces the number of troublesome race conditions. The new code will immediately remove swapped-in folios from the swap cache for `SWP_SYNCHRONOUS_IO` devices as a way of freeing the memory used for the swapped data. 

There is one interesting side effect of removing the swap-bypass code. In current kernels, large (multi-page) folios can only be swapped in intact if their reference count is one — only in the bypass case, in other words. Removal of the bypass feature makes it possible to swap in large folios from fast devices regardless of the reference count. 

Removal of swap bypass simplifies the swap-map management and makes it easier for the rest of the series to [coalesce swap-slot management](<https://lwn.net/ml/all/20251220-swap-table-p2-v5-14-8862a265a033@tencent.com>) into a small set of well-defined functions. Among other things, these functions are all folio-based, reducing the historical page orientation of the swap subsystem. All of those functions use a combination of the cluster lock and the folio lock to manage the swap cache. From there, it is just one more step to use those locks to control access to the swap map as well. 

Once the swap cache takes on the role of managing concurrency, there is only one last need for the `SWAP_HAS_CACHE` bit: marking swap slots that are allocated, but which have a reference count of zero. On the swap-out side, this situation is eliminated by immediately adding a folio to the swap cache once its slot has been assigned. At the other end, when pages are removed from the swap cache, swap slots with zero references are freed immediately. At that point, `SWAP_HAS_CACHE` is no longer needed; [this patch](<https://lwn.net/ml/all/20251220-swap-table-p2-v5-18-8862a265a033@tencent.com>) near the end of the series removes it. 

#### Removing the swap map

The work described above is, as of this writing, in the mm-unstable repository (and thus linux-next) and could be merged into the mainline as soon as the 7.0 release. But there is more to come. The [third phase of this](<https://lwn.net/ml/all/20260128-swap-table-p3-v2-0-fe0b67ef0215@tencent.com>) work is currently under review; this relatively short series eliminates the swap map entirely. 

Recall, from the previous installment, that the entries in the new swap table, which are simple `unsigned long` values, were the same as those stored in the XArray data structures in previous kernels. A value of zero indicates an empty slot. For a resident folio, the entry contains the folio address; for swapped folios, the entry contains the shadow information used to track which pages are quickly faulted back in from swap. The third phase changes the format of this table to support five different types of entries: 

  * A value of zero still indicates an empty slot. 
  * If bit zero is set, then this is a shadow entry for a swapped-out folio, but the upper part of the entry holds the reference count for this entry. The specific number of bits available for this count will vary depending on the architecture. 
  * If the bottom two bits are `10`, then the entry is for a folio that is resident in memory. As with shadow entries, the uppermost bits hold the reference count. To make room for that count, the page-frame number of the underlying page is stored rather than its address. 
  * A "pointer" entry is marked by setting the bottom three bits to `100`; pointers are not used in the current series. 
  * Setting the bottom four bits to `1000` marks a bad slot that should not be used. 

This organization takes the final remaining purpose for the swap map — tracking the reference counts — and shoehorns it into the swap table; that allows the swap map to be removed altogether. The result is a more compact memory representation and some significant memory savings; Song estimates that about 30% of the swap subsystem's metadata overhead is gone, saving 256MB of memory for a 1TB swap file. Until now, the kernel has maintained the swap map (tracking the status of slots in a swap file) and the swap cache (which tracks the pages that have been placed into swap) separately. The unification of those two data structures, Song says, reduces the amount of record-keeping overhead significantly, speeding the swap system overall. 

The new format can keep a larger reference count than the swap map can. For example, x86_64 systems will need 40 bits to hold the page-frame number, plus two for the resident-folio marker; that leaves 22 bits for the reference count. That size will be smaller on some other architectures (especially 32-bit systems) and, in any case, the possibility of overflow still exists. The complex system used to handle reference-count overflow in current kernels has been removed, though. Instead, if a reference count overflows, an array of `unsigned long` counts will be allocated for the entire cluster. 

The third phase is in its second revision. Thus far, neither version has received much in the way of review comments; that suggests that the removal of the swap map is not yet imminent. Even once this happens, though, the work is not done; Song has alluded to a later phase that will integrate the swapping limits from the memory controller into the swap table as well. So, just like the rest of the kernel, the swap subsystem is unlikely to be considered complete anytime soon.
