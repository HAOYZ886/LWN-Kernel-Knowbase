---
title: "Long-term get_user_pages() and truncate(): solved at last?"
url: https://lwn.net/Articles/796000/
date: "August 13, 2019"
category: "Memory management-get user pages"
author: "By Jonathan Corbet August 13, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 13, 2019

Technologies like [RDMA](<https://en.wikipedia.org/wiki/Remote_direct_memory_access>) benefit from the ability to map file-backed pages into memory. This benefit extends to persistent-memory devices, where the backing store for the file can be mapped directly without the need to go through the kernel's page cache. There is a fundamental conflict, though, between mapping a file's backing store directly and letting the filesystem code modify that file's on-disk layout, especially when the mapping is held in place for a long time (as RDMA is wont to do). The problem seems intractable, but there may yet be a solution in the form of [this patch set](<https://lwn.net/ml/linux-kernel/20190809225833.6657-1-ira.weiny@intel.com/>) (marked "V1,000,002") from Ira Weiny. 

The problems raised by the intersection of mapping a file (via `get_user_pages()`), persistent memory, and layout changes by the filesystem were the topic of [a contentious session](<https://lwn.net/Articles/787636/>) at the 2019 Linux Storage, Filesystem, and Memory-Management Summit. The core question can be reduced to this: what should happen if one process calls `[truncate()](<http://man7.org/linux/man-pages/man2/truncate.2.html>)` while another has an active `get_user_pages()` mapping that pins some or all of that file's pages? If the filesystem actually truncates the file while leaving the pages mapped, data corruption will certainly ensue. The options discussed in the session were to either fail the `truncate()` call or to revoke the mapping, causing the process that mapped the pages to receive a `SIGBUS` signal if it tries to access them afterward. There were passionate proponents for both options, and no conclusion was reached. 

Weiny's new patch set resolves the question by causing an operation like `truncate()` to fail if long-term mappings exist on the file in question. But it also requires user space to jump through some hoops before such mappings can be created in the first place. This approach comes from the conclusion that, in the real world, there is no rational use case where somebody might want to truncate a file that has been pinned into place for use with RDMA, so there is no reason to make that operation work. There is ample reason, though, for preventing filesystem corruption and for informing an application that gets into such a situation that it has done something wrong. 

#### Layout leases

The bulk of the patch set, though, is a recognition that the creation of long-term mapped kernel-mapped pages on persistent storage is not something that should be done lightly. Any user-space process that wants to set up such a mapping first has to convince the kernel that the associated issues have been thought about. That involves extending the existing lease mechanism in the kernel. 

The `F_SETLEASE` command for `[fcntl()](<http://man7.org/linux/man-pages/man2/fcntl.2.html>)` allows a process to obtain a "lease" on a file. Leases will not prevent other processes from carrying out most operations, but they will lead to the leaseholder receiving a notification (in the form of a signal) when another process is about to make a change to a file. The leaseholder then has a fixed amount of time to perform any necessary cleanup and remove its lease before the change is allowed to proceed. Leases, thus, do not guarantee exclusive access to a file, but they do give the leaseholder a window in which to prepare for and cope with any independent changes. 

Inside the kernel, there is a concept of a "layout lease" that is not exposed to user space. Layout leases are used with [pNFS](<http://www.pnfs.com/>) to arbitrate access with respect to operations that can change the on-disk layout of a file. The first step in Weiny's patch set is to make the `F_LAYOUT` lease type available to user space as well. A process that is mapping a file on persistent storage into kernel memory could use this type of lease to do the right thing when an independent operation changes the layout of a file. The patch set modifies the XFS and ext4 filesystems to break layout leases before performing a layout-changing operation in order to support this mode of operation. 

For the intended RDMA use case, though, the "right thing" — unmapping the file and letting the operation proceed — is not really an option. The memory in question is, almost certainly, under the control of a network interface for DMA operations, and unmapping it would be disruptive at best. So the second patch in the series adds a new `F_EXCLUSIVE` flag to indicate a lease that cannot be broken. A holder of such a lease need not worry about handling signals or coping with independent changes; any operation that would cause that to happen — `truncate()`, for example — will instead fail with an `ETXTBSY` error. 

The final step is to change `get_user_pages()` to require the existence of an exclusive layout lease before any pages can be mapped with the `FOLL_LONGTERM` flag (which indicates that the mapping is expected to be held for a long period of time). Applications that need to create such mappings will thus need to be changed to obtain the exclusive lease first. That might be seen as an ABI change except for the fact that, until the [final patch](<https://lwn.net/ml/linux-kernel/20190809225833.6657-20-ira.weiny@intel.com/>) in this series, such mappings have not been allowed at all. 

It's worth noting that the application need not continue to hold the lease after mapping the file; indeed, it need not keep the file open at all. While an exclusive layout lease does exist, no attempt to change the layout of the file will succeed. In the absence of the lease, layout changes will still fail if they involve any pages that have been pinned with `get_user_pages()`. So it may still be possible to call `truncate()` on a file with long-term pinned pages, but only as long as there is no layout lease in place and the operation does not affect those pages. This behavior fits the intended use case: registration of a file on persistent-memory storage with the kernel for RDMA operation. 

#### Accounting and the future

The remaining patches in this series are concerned with keeping track of what is going on. Should this code be merged, it is surely only a matter of time until some user complains about being unable to truncate a file and wondering just what is blocking the operation. Some new files added to `/proc` will provide information about which files have pages pinned and who is responsible for them. 

Given the heat that has surrounded this problem for years, the response to this patch set has been relatively muted. Dave Chinner has [expressed some reservations](<https://lwn.net/ml/linux-kernel/20190809235231.GC7777@dread.disaster.area/>) regarding the semantics of the newly user-visible layout leases, but he seems to be more concerned with nailing down the exact behavior than with the idea as a whole. Nobody has argued that the overall idea will not work. So it seems that the proposed solution might just be good enough to eventually find its way upstream.
