---
title: A FUSE implementation for famfs
url: https://lwn.net/Articles/1020170/
date: "May 8, 2025"
category: "Compute Express Link CXL; DAX; Filesystems-famfs"
author: "By Jake Edge May 8, 2025 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
May 8, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The [famfs](<https://github.com/cxl-micron-reskit/famfs/blob/master/README.md>) filesystem is meant to provide a shared-memory filesystem for large data sets that are accessed for computations by multiple systems. It was developed by John Groves, who led a combined filesystem and memory-management session at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF) to discuss it. The session was a follow-up to [the famfs session at last year's summit](<https://lwn.net/Articles/983105/>), but it was also meant to discuss whether the kernel's [direct-access](<https://docs.kernel.org/filesystems/dax.html>) (DAX) mechanism, which is used by famfs, could be replaced in the filesystem by using other kernel features. 

Groves said that he works for a company that makes memory; what it is trying to do is ""make bigger problems fit in memory through pools of disaggregated memory"" as an alternative to [sharding](<https://en.wikipedia.org/wiki/Shard_\(database_architecture\)>). He comes from a physics background where they would talk about two kinds of problems: those that fit in memory and those that do not. That's still true today, even though there is lots more memory available. 

For the most part, famfs is targeting read-only or read-mostly use cases, at least ""for sane people"". For crazy people, though, it is possible to handle shared-writable use cases; the overhead for ensuring coherency will not make users happy, however. It turns out that many analytics use cases simply need shared read-only access to the data once it is loaded. 

David Howells asked what he meant by "disaggregated"; Groves said he did not want to use the term "[CXL](<https://en.wikipedia.org/wiki/Compute_Express_Link>)", but that is the best example of disaggregated memory, which means it is not private to a single server. There are other ways to provide that, but CXL is the most practical choice currently. Linux can provide ways to access disaggregated memory, but it cannot manage it like regular system RAM. 

[ ![\[John Groves\]](https://static.lwn.net/images/2025/lsfmb-groves-sm.png) ](<https://lwn.net/Articles/1020436/>)

There are some other differences between system RAM and "special-purpose memory" (SPM), which is the category disaggregated memory falls into. System RAM is not shareable between systems, while SPM is. The [reliability, availability, and serviceability](<https://www.kernel.org/doc/html/latest/admin-guide/RAS/main.html>) (RAS) ""blast radius"" for system RAM includes the kernel, while, for SPM, it is only the applications that are affected by an outage, since the kernel does not directly use the memory. When using system RAM, no device abstraction is needed as the memory is simply presented as a contiguous range, but he would argue that any time two systems (or subsystems) need to agree on the handling of the contents of memory, a device abstraction is required. 

Famfs is ""an append-only log-structured filesystem"" with a log that is stored in memory. It prohibits truncate and delete operations, though filesystems can simply be destroyed if needed; that is done to avoid having to handle clients with stale metadata. He is currently working on two separate implementations: one is a kernel filesystem that he talked about last year and the other is a [Filesystem in Userspace](<https://github.com/libfuse/libfuse?tab=readme-ov-file#libfuse>) (FUSE) version. The latter was suggested in last year's session and he has it working, but had not yet posted it; a month later, on April 20, he [posted RFC patches for FUSE famfs](<https://lwn.net/ml/all/20250421013346.32530-1-john@groves.net/>). 

Despite the fact that CXL is not commercially available, at least without having connections in the industry, there are already users of famfs. He is ""getting, sometimes cagey, feedback from hyperscalers"" and analytics companies are also using it. Famfs allows users to rethink how much of the data set they can put in memory; putting a 64TB data frame into memory that can be shared among compute nodes is valuable even if the access is slower. No one likes to do sharding, Groves said, but they do it because they have to. 

The FUSE port of famfs adds two new FUSE messages. The first is `GET_FMAP`, which retrieves the full mapping from a file to its DAX extents that gets cached in system memory. That means that all active files can be handled quickly since their metadata is cached in memory. The other new message is `GET_DAXDEV`, which retrieves the DAX device information. It is used if the file map refers to DAX devices that are not yet known to the FUSE server. 

The famfs FUSE implementation in the kernel provides read, write, and [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) capabilities, along with fault handling. There are some small [patches for `libfuse`](<https://github.com/libfuse/libfuse/pull/1200>) to handle the new messages as well. Famfs disables the `READDIRPLUS` functionality, which returns [`stat()`](<https://www.man7.org/linux/man-pages/man2/stat.2.html>) information for multiple files, because it does not provide the needed file-map information. 

He has been asked if famfs could use memfd (from [`memfd_create()`](<https://man7.org/linux/man-pages/man2/memfd_create.2.html>)) or [guest_memfd](<https://lwn.net/Articles/949277/>) instead of DAX; he believes the answer is ""probably not"", but wanted to discuss it with attendees. Using a DAX device allows errors with the memory to be returned, which famfs uses, though it is not yet plumbed into the FUSE version. Previously, he supported both persistent memory (pmem) and DAX devices as the backing store for famfs, but it was confusing, so he dropped pmem support. Using memfd is not workable, since it operates on system RAM and not SPM, he said. 

There is some information out there about using guest_memfd with DAX, Groves said, or maybe it is all just an AI hallucination. David Hildenbrand said that there were ideas floating around about that; guest_memfd is just a file, not a filesystem, but perhaps the allocator could be changed to use SPM, instead of system RAM. Groves thought that would defeat the use case for famfs; it is important that the SPM be accessible from all of the systems in a cluster, which a DAX device can provide and it did not sound like guest_memfd could do so. 

There are a number of different kinds of use cases that benefit from the famfs approach, Groves said. For example, providing parallel access to an enormous [RocksDB](<https://rocksdb.org/>) database. Another, that kernel developers may not be aware of, is data frames for [Apache Arrow](<https://arrow.apache.org/>), which provides a memory-efficient layout for data that makes it easily accessible from CPUs and GPUs for analytics workloads. Famfs can be used for ""ginormous Arrow data frames that multiple nodes want to `mmap()`"" with full isolation between the different files in the filesystem. 

Matthew Wilcox said that he was not going to repeat his criticisms of CXL, which are well-known, he said and Groves acknowledged with a chuckle. The approach taken with famfs is reasonable for an experiment, Wilcox continued, but his concern is that there are already several shared-storage filesystems in the kernel, such as [OCFS2](<https://www.kernel.org/doc/html/latest/filesystems/ocfs2.html>) and [GFS2](<https://www.kernel.org/doc/html/latest/filesystems/gfs2.html>), why not adapt one of those for the famfs use cases? Groves said he is not a cluster-filesystem expert but the main difference is that famfs is backed by memory not storage; reads are done in increments of cache lines, not pages. If, for example, the application is chasing pointers through the data, each new access just requires a read of a cache line. 

As the session wound down, there was some discussion of problems that Groves had reported that resulted in spurious warnings from the kernel. He said that he thinks those problems are fixable, and that pmem does not have the same kinds of problems. He hopes to be able to apply the pmem solution to famfs.
