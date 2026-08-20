---
title: "Memory-management: tiered memory, huge pages, and EROFS"
url: https://lwn.net/Articles/984839/
date: "August 15, 2024"
category: "Filesystems-EROFS; Memory management-Huge pages; Memory management-Tiered-memory systems"
author: "By Jonathan Corbet August 15, 2024"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 15, 2024

The kernel's memory-management developers have been busy in recent times; it can be hard to keep up with all that has been happening in this core area. In an attempt to catch up, here is a look at recent work affecting tiered-memory systems, underutilized huge pages, and duplicated file data in the Enhanced Read-Only Filesystem (EROFS). 

#### Promotion and demotion

Tiered-memory systems are built with multiple types of memory that have different performance characteristics; the upper tiers are usually faster, while lower tiers are slower but more voluminous. To make the best use of these memory tiers, the system must be able to optimally place each page. Heavily used pages should normally go into the fast tiers, while memory that is only occasionally used is better placed in the slower tiers. Since usage patterns change over time, the optimal placement of memory will also change; the system must be able to move pages between tiers based on current usage. Promoting and demoting pages in this way is one of the biggest challenges in tiered-memory support. 

Promotion is usually the easier side of the problem; it is not hard for the system to detect when memory is being accessed and move it to a faster tier. In current kernels, though, this migration only works for memory that has been mapped into a process's address space; the machinery requires that memory be referred to by a virtual memory area (VMA) to function. As a result, heavily used memory that is not currently mapped will not be promoted. 

This situation comes about for page-cache pages that are being accessed by way of system calls (such as [`read()`](<https://man7.org/linux/man-pages/man2/read.2.html>) and [`write()`](<https://man7.org/linux/man-pages/man2/write.2.html>)), but which are not mapped into any address space. Memory-access speed can be just as important for such pages, though, so this inability to promote them can hurt performance. 

[This patch series](<https://lwn.net/ml/all/20240803094715.23900-1-gourry@gourry.net>) from Gregory Price is an attempt to address that problem. The migration code in current kernels ([`migrate_misplaced_folio_prepare()`](<https://elixir.bootlin.com/linux/v6.11-rc2/source/mm/migrate.c#L2551>) in particular) needs to consult the VMA that maps a given folio (set of pages) prior to migration; if that folio is both shared and mapped with execute permission, then the migration will not happen. Pages that are not mapped at all, though, cannot meet that condition, so the absence of a VMA just means that this check need not be performed. With that change (and a couple of other adjustments) in place, it is simply a matter of adding an appropriate call in the swap code to migrate folios from a lower to a higher tier when they are referenced. 

A kernel that is trying to appropriately place memory will always be running a bit behind the game; it cannot detect a changed access pattern without first watching the new pattern play out. Sometimes, though, an application will know that it will be shifting its attention from one range of memory to another. Informing the kernel of that fact might help the system ensure that memory is in the best location before it is needed; at least, that is the intent behind [this patch](<https://lwn.net/ml/all/20240801075610.12351-1-zhang.renze@h3c.com>) from "BiscuitOS Broiler". 

Quite simply, this patch adds two new operations to the [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) system call. They are called `MADV_DEMOTE` and `MADV_PROMOTE`; they do exactly what one would expect. An application can use these operations to explicitly request the movement of memory between tiers in cases where it knows that the access pattern is about to change. 

There is nothing technically challenging about this work, but it is also not clear that it is necessary. The kernel already provides a system call, [`migrate_pages()`](<https://man7.org/linux/man-pages/man2/migrate_pages.2.html>), that can be used to move pages between tiers; David Hildenbrand [asked](<https://lwn.net/ml/all/a530a614-41a3-4bd1-a2df-40ade7eca002@redhat.com/>) why `migrate_pages()` is not sufficient in this case. The [answer](<https://lwn.net/ml/all/3a5785661e1b4f3381046aa5e808854c@h3c.com/>) seems to be that `madvise()` is found in the C library, but the wrapper for `migrate_pages()` is in the extra `libnuma` library instead. As Hildenbrand [answered](<https://lwn.net/ml/all/bffe178c-bd97-4945-898e-97ba203f503e@redhat.com/>), that is not a huge impediment to its use. So, while making this feature available via `madvise()` might be convenient for some users, that convenience seems unlikely to be enough to justify adding this new feature to the kernel. 

#### Reclaiming underutilized huge pages

The use of huge pages can improve application performance, by reducing both the usage of the system's translation lookaside buffer (TLB) and memory-management overhead in the kernel. But huge pages can suffer from internal fragmentation; if only a small part of the memory within a huge page is actually used, the resulting waste can be significant. The corresponding increase in memory use has inhibited the adoption of huge pages in many settings that would otherwise benefit from them. 

One way to get the best of both worlds might be to actively detect huge pages that are not fully used, split them apart into base pages, then reclaim the unused base pages; that is the objective of [this patch series](<https://lwn.net/ml/all/20240813120328.1275952-1-usamaarif642@gmail.com>) from Usama Arif. It makes two core changes to the memory-management subsystem aimed at recovering memory that is currently wasted due to internal fragmentation. 

The first of those changes takes effect whenever a huge page is split apart and mapped at the base-page level, as often happens even in current kernels. As things stand now, splitting a huge page will leave the full set of base pages in its wake, meaning that the amount of memory in use does not change. But, if the huge page is an anonymous (user-space data) page, any base pages within it that have not been used will only contain zeroes. Those base pages can be replaced in the owning process's page tables with references to the shared zero page, freeing that memory. Arif's patch set makes that happen by checking the contents of base pages during the splitting process and freeing any pages found to hold only zeroes. 

That will free underutilized memory when a page is being split, which is a start. It would work even better, though, if the kernel could actively find underutilized huge pages and split them when memory is tight; that is the objective of the second change in Arif's patch set. 

A huge page, as represented by a folio within the kernel, can at times be partially mapped, meaning that not all of the base pages within the huge page have been mapped in the owning process's page tables. When a fully mapped folio is partially unmapped for any reason, the folio is added to the "deferred split list". If, at some later point, the kernel needs to find some free memory, it will attempt to split the folios on the deferred list, then work to reclaim the base pages within each of them. 

Arif's patch set causes the kernel to add _all_ huge pages to the deferred list whenever they are either faulted in or created from base pages by the `khugepaged` thread. When memory gets tight and the deferred list is processed, these huge pages (which are probably still fully mapped) will be checked for zero-filled base pages; if the number of such pages exceeds a configurable threshold, the huge page will be split and all of those zero-filled base pages will be immediately freed. If the threshold is not met, instead, the page will be considered to be fully used and removed from the deferred list. 

It is worth noting that the threshold is an absolute number; for the tests mentioned in the cover letter it was set to 409, which is roughly 80% of a 512-page huge page. This mechanism means that, while this feature can split underutilized PMD-sized huge pages implemented by the processor, it will not be able to operate on smaller, multi-size huge pages implemented in software. On systems using PMD-sized huge pages, though, the results reported in the cover letter show that this change can provide the performance benefits that come from enabling transparent huge pages while clawing back most of the extra memory that would otherwise be wasted. 

#### Page-cache deduplication for EROFS

Surprisingly often, a system's memory will contain multiple pages containing the same data. When this happens with anonymous pages, the [kernel samepage merging](<https://docs.kernel.org/admin-guide/mm/ksm.html>) feature can perform deduplication, recovering some memory (albeit with some security concerns). The situation with file-backed pages is harder, though. Filesystems that can cause a single file to appear with multiple names and inodes (as can happen with Btrfs snapshots or in filesystems that provide a "reflink" feature) are one case in point; if more than one name is used, multiple copies of a file's data can appear in the page cache. This can also happen in the mundane cases where files contain the same data; container images can duplicate data in this way. 

The problem with deduplicating such pages is that each page in the page cache must refer back to the file from which it came; there is no concept in the kernel of a page coming from multiple files. If a page is written to, or if a file changes by some other means, the kernel has to do the right thing at all levels. So those duplicate pages remain duplicated. 

Hongzhen Luo has come up with [a solution](<https://lwn.net/ml/all/20240731080704.678259-1-hongzhen@linux.alibaba.com>) for the [EROFS](<https://docs.kernel.org/filesystems/erofs.html>) filesystem, though — at the file level, at least. EROFS is a read-only filesystem, so the problems that come from possible changes to its files do not arise here. 

An EROFS filesystem can be created with a special extended attribute, called `trusted.erofs.fingerprint`, attached to each file; the content of that attribute is a hash of the file's contents. When a file in the filesystem is opened for reading, the hash will be stored in an [XArray](<https://docs.kernel.org/core-api/xarray.html>)-based data structure, associated with the file's inode. Anytime another file is opened, its hash is looked up in that data structure; if there is a match, the open is rerouted to the inode of the file that was opened first. 

This mechanism can result in a number of processes holding file descriptors to different files on disk that all refer to a single file within the kernel. Since the files have the same contents, though, this difference is not visible to user space (though an [`fstat()`](<https://man7.org/linux/man-pages/man2/stat.2.html>) call might return a surprising inode number). Within the kernel, redirecting file descriptors for multiple identical files to a single file means that only one copy of that file's contents needs to be stored in the page cache. 

The benchmark results included with the series show a significant reduction in memory use for a number of different applications. Since this feature is contained entirely within the EROFS filesystem, it seems unlikely to run into the sorts of challenges that often await core memory-management patches. Deduplication of file-backed data in the page cache remains a hard problem in the general case, but it appears to have been at least partially solved for this one narrow case.
