---
title: "Modernizing swapping: virtual swap spaces"
url: https://lwn.net/Articles/1059201/
date: "February 19, 2026"
category: "Memory management-Swapping"
author: "By Jonathan Corbet February 19, 2026"
---

By **Jonathan Corbet**  
February 19, 2026

The kernel's unloved but performance-critical swapping subsystem has been undergoing multiple rounds of improvement in recent times. Recent articles have described [the addition of the swap table](<https://lwn.net/Articles/1056405/>) as a new way of representing the state of the swap cache, and [the removal of the swap map](<https://lwn.net/Articles/1057102/>) as the way of tracking swap space. Work in this area is not done, though; [this series from Nhat Pham](<https://lwn.net/ml/all/20260208215839.87595-1-nphamcs@gmail.com>) addresses a number of swap-related problems by replacing the new swap table structures with a single, virtual swap space. 

#### The problem with swap entries

As a reminder, a "swap entry" identifies a slot on a swap device that can be used to hold a page of data. It is a 64-bit value split into two fields: the device index (called the "type" within the code), and an offset within the device. When an anonymous page is pushed out to a swap device, the associated swap entry is stored into all page-table entries referring to that page. Using that entry, the kernel can quickly locate a swapped-out page when that page needs to be faulted back into RAM. 

The "swap table" is, in truth, a set of tables, one for each swap device in the system. The transition to swap tables has simplified the kernel considerably, but the current design of swap entries and swap tables ties swapped-out pages firmly to a specific device. That creates some pain for system administrators and designers. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

As a simple example, consider the removal of a swap device. Clearly, before the device can be removed, all pages of data stored on that device must be faulted back into RAM; there is no getting around that. But there is the additional problem of the page-table entries pointing to a swap slot that no longer exists once the device is gone. To resolve that problem, the kernel must, at removal time, scan through all of the anonymous page-table entries in the system and update them to the page's new location. That is not a fast process. 

This design also, as Pham describes, creates trouble for users of the [zswap subsystem](<https://docs.kernel.org/admin-guide/mm/zswap.html>). Zswap works by intercepting pages during the swap-out process and, rather than writing them to disk, compresses them and stores the result back into memory. It is well integrated with the rest of the swapping subsystem, and can be an effective way of extending memory capacity on a system. When the in-memory space fills, zswap is able to push pages out to the backing device. 

The problem is that the kernel must be able to swap those pages back in quickly, regardless of whether they are still in zswap or have been pushed to slower storage. For this reason, zswap hides behind the index of the backing device; the same swap entry is used whether the page is in RAM or on the backing device. For this trick to work, though, the slot in the backing device must be allocated at the beginning, when a page is first put into zswap. So every zswap usage must include space on a backing device, even if the intent is to never actually store pages on disk. That leads to a lot of wasted storage space and makes zswap difficult or impossible to use on systems where that space is not available to waste. 

#### Virtual swap spaces

The solution that Pham proposes, as is so often the case in this field, is to add another layer of indirection. That means the replacement of the per-device swap tables with a single swap table that is independent of the underlying device. When a page is added to the swap cache, an entry from this table is allocated for it; the swap-entry type is now just a single integer offset. The table itself is an array of `swp_desc` structures: 

```
struct swp_desc {
            union {
                swp_slot_t         slot;
                struct zswap_entry * zswap_entry;
            };
            union {
                struct folio *     swap_cache;
                void *             shadow;
            };
            unsigned int               swap_count;
            unsigned short             memcgid:16;
            bool                       in_swapcache:1;
            enum swap_type             type:2;
        };
```

The first union tells the system where to find a swapped-out page; it either points to a device-specific swap slot or an entry in the zswap cache. It is the mapping between the virtual swap slot and a real location. The second union contains either the location of the page in RAM (or, more precisely, its folio) or the shadow information used by the memory-management subsystem to track how quickly pages are faulted back in. The `swap_count` field tracks how many page-table entries refer to this swap slot, while `in_swapcache` is set when a page is assigned to the slot. The control group (if any) managing this allocation is noted in `memcgid`. 

The `type` field tells the kernel what type of mapping is currently represented by this swap slot. If it is `VSWAP_SWAPFILE`, the virtual slot maps to a physical slot (identified by the `slot` field) on a swap device. If, instead, it is `VSWAP_ZERO`, it represents a swapped-out page that was filled with zeroes that need not be stored anywhere. `VSWAP_ZSWAP` identifies a slot in the zswap subsystem (pointed to by `zswap_entry`), and `VSWAP_FOLIO` is for a page (indicated by `swap_cache`) that is currently resident in RAM. 

The big advantage of this arrangement is that a page can move easily from one swap device to another. A zswap page can be pushed out to a storage device, for example, and all that needs to change is a pair of fields in the `swp_desc` structure. The slot in that storage device need not be assigned until a decision to push the page out is made; if a given page is never pushed out, it will not need a slot in the storage device at all. If a swap device is removed, a bunch of `swp_desc` entries will need to be changed, but there will be no need to go scanning through page tables, since the virtual swap slots will be the same. 

The cost comes in the form of increased memory usage and complexity. The swap table is one 64-bit word per swap entry; the `swp_desc` structure triples that size. Pham points out that the added memory overhead is less than it seems, since this structure holds other information that is stored elsewhere in current kernels. Still, it is a significant increase in memory usage in a subsystem whose purpose is to make memory available for other uses. This code also shows performance regressions on various benchmarks, though those have improved considerably from previous versions of the patch set. 

Still, while the value of this work is evident, it is not yet obvious that it can clear the bar for merging. Kairui Song, who has done the bulk of the swap-related work described in the previous articles, has [expressed concerns](<https://lwn.net/ml/all/CAMgjq7AQNGK-a=AOgvn4-V+zGO21QMbMTVbrYSW_R2oDSLoC+A@mail.gmail.com/>) about the memory overhead and how the system performs under pressure. Chris Li also [worries about the overhead](<https://lwn.net/ml/all/CACePvbVvzh8PcF47hz+MfFu3tta5vh3oD+WpGxEL_-NrzYZG3Q@mail.gmail.com/>) and said that the series is too focused on improving zswap at the expense of other swap methods. So it seems likely that this work will need to see a number of rounds of further development to reach a point where it is more widely considered acceptable. 

#### Postscript: swap tiers

There is a separate project that appears to be entirely independent from the implementation of the virtual swap space, but which might combine well with it: the [swap tiers patch set](<https://lwn.net/ml/all/20260217000950.4015880-1-youngjun.park@lge.com>) from Youngjun Park. In short, this series allows administrators to configure multiple swap devices into tiers; high-performance devices would go into one tier, while slower devices would go into another. The kernel will prefer to swap to the faster tiers when space is available. There is a set of control-group hooks to allow the administrator to control which tiers any given group of processes is allowed to use, so latency-sensitive (or higher-paying) workloads could be given exclusive access to the faster swap devices. 

A virtual swap table would clearly complement this arrangement. Zswap is already a special case of tiered swapping; Park's infrastructure would make it more general. Movement of pages between tiers would become relatively easy, allowing cold data to be pushed in the direction of slower storage. So it would not be surprising to see this patch series and the virtual swap space eventually become tied together in some way, assuming that both sets of patches continue to advance. 

In general, the kernel's swapping subsystem has recently seen more attention than it has received in years. There is clearly interest in improving the performance and flexibility of swapping while making the code more maintainable in the long run. The days when developers feared to tread in this part of the memory-management subsystem appear to have passed.
