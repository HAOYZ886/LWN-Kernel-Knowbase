---
title: Preserving the mobility of ZONE_MOVABLE
url: https://lwn.net/Articles/843326/
date: "January 22, 2021"
category: "Memory management-get user pages"
author: "By Jonathan Corbet January 22, 2021"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 22, 2021

Memory fragmentation has long been a problem for Linux systems, to the point that, for years, finding even two physically contiguous pages was an uncertain affair. That said, the situation has improved considerably in the last decade or so thanks to a number of changes implemented by the memory-management developers. One of those changes is the creation of "movable" memory zones where pages can be relocated if need be. All that work is for nothing, though, if somebody comes along and pins down a page in one of these movable zones. [This patch set](<https://lwn.net/ml/linux-kernel/20210122033748.924330-1-pasha.tatashin@soleen.com/>) from Pavel Tatashin seeks to prevent that from happening, but may risk creating problems elsewhere. 

The Linux kernel allocates and frees pages of memory at a high rate; at any given time, there is no real way to predict which pages will be free, and which will be in use. Those in-use pages have an annoying tendency to be spread out across physical memory, making it hard to find a range of free, physically contiguous pages. The increasing use of huge pages, though, has increased the demand for contiguous regions — regions that will not be available if the kernel does not make an active effort to create them. 

Contemporary systems can also face a related problem: hotpluggable memory. While there are machines with memory that can be physically yanked out of the box, hotplugging is more commonly used in the context of virtualization. Virtual machines experiencing memory thrashing can have more memory "plugged" into them by the hypervisor, while those that are not fully using the memory they have can be made to give some back to the system. Unplugging memory can be seen as a special case of the "large, physically contiguous region" problem — the whole range of memory must be made to be free so it can be unplugged without data loss — with the added constraint that the region must be located at a specific address. It is fair to say that this rarely happens by chance. 

In both cases, the kernel can respond by migrating in-use pages that are in the way of creating the needed region — if the pages can, in fact, be migrated. In practice, pages allocated for user space can be moved at will; they are always accessed through the page tables, so moving them is just a matter of changing the appropriate page-table entries to point to the new location, and user space will never be the wiser. Pages allocated for use by the kernel, instead, tend not to be movable. Technically they, too, are accessed via page tables, but those page-table entries generally cannot be changed, so the pages must remain where they are. 

[For those who are curious about why the page-table entries cannot be changed, there are (at least) a couple of reasons. One is that those pages are usually accessed via the kernel's direct mapping, which is a mirror of physical memory and cannot be remapped in any practical way. The other is that moving a page requires invalidating the page-table entry while the move is in progress, and the kernel isn't prepared for the page faults that would result if the page is accessed during that time.] 

It only takes one immobile page to ruin a block of memory; it takes relatively few of them to make large contiguous regions hard to create. So if these non-movable pages are placed randomly in memory, fragmentation is certain to happen. 

The kernel already divides memory into a number of zones with differing characteristics, including their addressability for DMA operations and which NUMA node they are on; [the full list of zone types](<https://elixir.bootlin.com/linux/v5.11-rc4/source/include/linux/mmzone.h#L348>) is found in `<linux/mmzone.h>`. The zone known as `ZONE_MOVABLE`, in particular, is meant to be used solely for allocations of movable pages. By segregating movable and non-movable allocations, the kernel tries to avoid the problem of single, badly placed, non-movable pages. Within `ZONE_MOVABLE`, it should be possible to move pages around (or simply reclaim them) to create larger regions as necessary. 

This all works fairly well until some rude process comes along and pins down a page in `ZONE_MOVABLE`. This can happen, for example, when I/O is performed directly to or from user-space memory; moving a page while a device is performing I/O to it tends to have unfortunate side effects. Often, this pinning is of short duration and tends not to be a big problem. At times, though, this pinning can last for long periods of time — weeks, for example. Memory buffers used for RDMA are often implicated in this sort of antisocial behavior, but there are other examples as well. A virtually contiguous user-space buffer may, of course, be widely scattered across physical memory, so pinning one of these buffers for long periods can create equally widely scattered problems. 

The problems with long-term pinning have been understood for a while. The kernel community's grasp of the solutions tends to be a bit more fuzzy, with the result that have been many discussions over the years aimed at improving the situation. One outcome from this work has been [the addition of some infrastructure](<https://lwn.net/Articles/807108/>) to indicate when pages are being pinned for a long time. Not much has been done with regard to what the kernel should do when confronted with a long-term pin, but knowing about it is a starting point. 

Tatashin's patch set aims to use that information for a specific goal: preventing the pinning of pages in `ZONE_MOVABLE`. To that end, it builds on work that was already done to deal with a related problem: non-movable pages in areas designed for the [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>). Specifically, in current kernels, a call to pin pages in memory assigned to CMA will cause an attempt to migrate those pages out of the CMA area before the pin takes effect. 

Tatashin starts by fixing a couple of problems with that code, one of which being that it will happily migrate pages into `ZONE_MOVABLE` before pinning them. The migration code is now instructed to avoid that zone. The other is that, in current kernels, a failed attempt to move a page is met with a sort of kernel-space shrug of the shoulders; the page will be pinned anyway, even though it is stuck in the CMA area. The patch set changes this behavior; if a page cannot be moved to a more appropriate location, the attempt to pin it will fail. 

Then, this mechanism is extended to operate on pages in `ZONE_MOVABLE` as well. If the kernel tries to pin pages in `ZONE_MOVABLE`, an attempt will be made to migrate them first; if that attempt fails, the pinning will not be allowed. 

These changes should help to keep `ZONE_MOVABLE` true to its name. That should allow operations like the creation of huge pages to succeed more often and remove impediments to the hot-unplugging of memory. On the other hand, these changes increase the chances that pinning operations will fail; when that happens, the failure will almost certainly be propagated directly to user space. That could be the cause of some disgruntlement. 

Whether this is a problem that will arise in practice is far from clear; in theory, now that pages cannot be pinned in `ZONE_MOVABLE`, migration failures should be rare. But this is the sort of change that can cause complaints years later when it turns up in an enterprise kernel. It would appear that the memory-management developers are not overly concerned about this possibility. The patch set has been through seven revisions and appears to be getting closer to the point where it can be accepted; the seventh version was mostly concerned with adding Reviewed-by tags to the series. So chances are that `ZONE_MOVABLE` will come closer to living up to its name in the near future.
