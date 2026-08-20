---
title: "Large-folio support for shmem and tmpfs"
url: https://lwn.net/Articles/974630/
date: "May 24, 2024"
category: "Filesystems-tmpfs"
author: "By Jonathan Corbet May 24, 2024 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 24, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The kernel contains a pair of related filesystems that, among other things, can be used for shared-memory applications; shmem is an internal mechanism used within the kernel, while the tmpfs filesystem is mounted and accessible from user space. As is the case elsewhere in the kernel, these subsystems would benefit from the addition of large-folio support. During a joint storage, filesystem, and memory-management session at the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Daniel Gomez talked about the work he is doing to add that support. 

Gomez started by saying that he had [posted a patch series](<https://lwn.net/ml/linux-kernel/20240515055719.32577-1-da.gomez@samsung.com/>) for shmem and tmpfs. It will cause a large folio to be allocated in response to a sufficiently large [`write()`](<https://man7.org/linux/man-pages/man2/write.2.html>) or [`fallocate()`](<https://man7.org/linux/man-pages/man2/fallocate.2.html>) call; variable sizes, up to the PMD size (2MB on x86) are supported. The patch implements block-level up-to-date tracking, which is needed to make the `SEEK_DATA` and `SEEK_HOLE` [`lseek()`](<https://man7.org/linux/man-pages/man2/lseek.2.html>) options work properly. Baolin Wang has also posted [a patch set](<https://lwn.net/ml/linux-kernel/cover.1714978902.git.baolin.wang@linux.alibaba.com/>) adding multi-size transparent huge page (mTHP) support to shmem. 

[![\[Daniel Gomez\]](https://static.lwn.net/images/conf/2024/lsfmm/DanielGomez-sm.png)](<https://lwn.net/Articles/974632/>) David Hildenbrand said that the biggest challenge in this work may be that many systems are configured to run without swap space. The shmem subsystem works in a weird space that is sometimes like anonymous memory, and sometimes like the page cache; that can lead to situations where the system is unable to reclaim memory. Using large folios in shmem, he said, could lead to the kernel wasting its scarce huge pages in mappings where they will not actually be used. 

Returning to his presentation, Gomez said that his current work only applies to the `write()` and `fallocate()` paths. But there is also a need to update the `read()` path. That can be managed by allocating huge pages depending on the size of the read request, but it is also worth considering whether readahead should be taken into account here. Then, there is the swap path; large folios are not currently enabled there, so they will be split if targeted by reclaim. With better up-to-date tracking, though, the swap path can perhaps be improved as well. Finally, he is also looking at the [`splice()`](<https://man7.org/linux/man-pages/man2/splice.2.html>) path; currently, if a large folio is fed to `splice()`, it will be split into base pages. 

When making significant changes to a heavily used subsystem like this, one needs to be worried about creating regressions. Gomez said that he has a set of machines running [kdevops](<https://github.com/mcgrof/kdevops>) tests, and the 0day robot has been testing his work as well. He is not sure what performance testing is being run; he did say that tmpfs is being outperformed by the XFS filesystem, and large-folio support makes the problem worse. The cause is currently a mystery. Hildenbrand said that, if the use of large folios is causing the memory-management subsystem to perform compaction, that could kill any performance benefit that would otherwise accrue. 

Gomez concluded by saying that, in the future, he plans to work on extending the swap code to handle large folios. He needs better ways to stress the swap path, and would appreciate hearing from anybody who can suggest good tests.
