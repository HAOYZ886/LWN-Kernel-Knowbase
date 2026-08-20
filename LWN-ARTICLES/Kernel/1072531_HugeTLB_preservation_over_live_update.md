---
title: HugeTLB preservation over live update
url: https://lwn.net/Articles/1072531/
date: "May 15, 2026"
category: Kexec handover; Virtualization
author: "By Jonathan Corbet May 15, 2026 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 15, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Recent times have seen a lot of effort put into the implementation of the [kexec handover and live update orchestrator](<https://lwn.net/Articles/1033364/>) features in the Linux kernel. But that work is not yet complete. At the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Pratyush Yadav led a memory-management-track session on adding the ability to preserve [hugetlbfs](<https://docs.kernel.org/admin-guide/mm/hugetlbpage.html>)-provided memory during the live-update process. 

The use case for live updates, Yadav began, is to replace the host kernel on a running system without disturbing the virtual machines running under that kernel. The [kexec handover](<https://docs.kernel.org/core-api/kho/index.html>) machinery marks regions of memory to be preserved during the kernel change; it is an in-kernel interface with no user-space API. Once memory has been marked, the [`kexec_load()` system call](<https://www.man7.org/linux/man-pages/man2/kexec_load.2.html>) is invoked to boot the new kernel, after which the preserved memory is restored to its owning process. The [live update orchestrator](<https://docs.kernel.org/core-api/liveupdate.html>) provides the user-space interface to this feature, letting users mark resources to preserve and restore them on the new kernel. 

[![\[Pratyush Yadav\]](https://static.lwn.net/images/conf/2026/lsfmm/PratyushYadav-sm.png)](<https://lwn.net/Articles/1072536/>) Kexec handover and the live update orchestrator were merged during the 6.19 development cycle, with limited support for the preservation of files in memory-based filesystems. It can handle memfds (anonymous files created with [`memfd_create()`](<https://man7.org/linux/man-pages/man2/memfd_create.2.html>)) backed by shared memory, but not much else. That feature can be used to preserve the contents of a virtual machine, but running virtual machines in memfds is relatively inefficient. A virtual machine placed in a small number of huge pages (perhaps of the 1GB variety) from the hugetlbfs subsystem will run more efficiently, but that memory will not, yet, survive a live update. 

Yadav's goals are to make it possible to carry those virtual machines (and their associated memory) through the live-update process. Preferably, it would be possible to obtain hugetlbfs pages by way of a special memfd, so that mounting hugetlbfs itself would not be necessary. The intent is to minimize the amount of preserved state; everything that is preserved becomes a part of the kernel's ABI, so less of it is better. He is also working to minimize the number of changes to hugetlbfs itself. 

The feature works by first freezing the contents of the huge pages to be preserved, so that changes are not lost during the update process. There are two ways of doing that; the first would be to add a flag to the associated hugetlbfs inodes, then check that flag when changes are about to be made. The shared-memory filesystem works this way. In addition, the relevant memory would be pinned to prevent migration or compaction during the update. This option can work, but it is seen as a bit of a hack by some developers. The alternative is to make the filemap code aware of freezing by way of a new address-space flag; once again, the flag would be checked before allowing modifications. There are various details that have to be managed to make this option work correctly, meaning that the virtual filesystem layer has to be aware of the freezing process. 

After the pages are frozen, some metadata about each — its size and position — is recorded. Both the frozen pages and the metadata are marked for preservation. Then the update can proceed. 

Once the new kernel is running, a new hugetlbfs-backed memfd is created, and each of its component huge pages is placed back into use, with the hugetlbfs state being updated accordingly. Control-group charging for the allocated memory is done, and the new pages are added to the page cache. At that point, from the point of view of memory preservation, the update process is complete. 

There is a bit of a complication regarding huge-page allocation, Yadav said. Hugetlbfs works by preallocating a set number of huge pages during the boot process. The restoration of a hugetlbfs-backed virtual machine will also allocate the necessary huge pages. In the original kernel, those pages were allocated from hugetlbfs; in the new kernel, instead, they are allocated separately. That can lead to over-allocation of huge pages in general. The solution is to count the number of preserved huge pages when the new kernel is booting, then to reduce the number preallocated by hugetlbfs accordingly. 

An as-yet unsolved problem is the interaction of this feature with the [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>). If the original huge pages were obtained from CMA, the new pages need to be inserted back into CMA after the update, but CMA does not offer any way of extending its memory zone. So the current patches just disable the use of hugetlbfs with CMA if live update is enabled. Someday it will be possible to preserve the state of CMA as well, but that does not happen with the current version of this work. 

Yadav closed with a status summary, saying that [an RFC patch set](<https://lwn.net/ml/all/20251206230222.853493-1-pratyush@kernel.org/>) had been posted in December 2025. As part of that work, he had found a number of problems with kexec handover that required [some significant infrastructure work](<https://lwn.net/ml/all/20260429133928.850721-1-pratyush@kernel.org/>) to fix. He will soon be posting an updated patch set, once he has responded to the feedback he has received. 

The only question came from Mike Rapoport, who wondered if making CMA pages movable would help in its integration with kexec handover. Yadav said that such a change had a high chance of breaking things and probably would not help much.
