---
title: The NOVA filesystem
url: https://lwn.net/Articles/754505/
date: "May 18, 2018"
category: "Filesystems-Nonvolatile memory"
author: "By Jake Edge May 18, 2018 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 18, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

At the 2018 Linux Storage, Filesystem, and Memory-Management Summit, Andiry Xu presented the [NOVA filesystem](<https://lwn.net/Articles/729812/>), which he is trying to get into the upstream kernel. Unlike existing kernel filesystems, NOVA exclusively targets non-volatile main memory (NVMM) rather than traditional block devices (disks or SSDs). In fact, it does not use the kernel's block layer at all and instead uses persistent memory mapped directly into the kernel address space. 

Xu compared NOVA to versions of the ext4 and XFS filesystems with support for the [DAX direct-access mechanism](<https://lwn.net/Articles/717953/>). With those, only the filesystem data bypasses the page cache; the metadata still goes through the page cache. In addition, those filesystems have a much higher latency for append operations. There is also a write amplification effect. All of that makes for high journaling overhead, he said. 

[ ![\[Andiry Xu\]](https://static.lwn.net/images/2018/lsf-xu-sm.jpg) ](<https://lwn.net/Articles/754533/>)

Beyond that, there are scalability issues for those filesystems on NVMM. He ran some tests on high-end multicore hardware to compare NOVA and tmpfs to the DAX modes of ext4 and XFS. In his tests, he emulated NVMM with RAM, since it is difficult to actually get NVMM devices at this point. In general, only tmpfs and NOVA scale reasonably—the other filesystems contend for various locks and semaphores—though there is still room for NOVA to improve as only tmpfs scaled reasonably for one of the tests. 

Support for huge pages is difficult for DAX filesystems, Xu said. Huge pages require that the physical address is aligned on a huge-page-size boundary and that the memory is physically contiguous, but memory allocated by filesystems does not necessarily conform to those requirements. Dave Chinner said that XFS has an inode option to support huge-page use; another attendee said that ext4 has an analogous feature but it can only support 2MB huge pages, not 1GB. 

Xu pointed attendees at the 2016 [NOVA paper [PDF]](<http://cseweb.ucsd.edu/~swanson/papers/FAST2016NOVA.pdf>) for more information, but gave a quick overview of some of NOVA's features. It is a log-structured filesystem that is designed for NVMM. It has per-inode logging that contains only the metadata changes; the log points off to changes to the actual data. It uses a radix tree for block mappings and is copy on write (CoW) for its file data. 

NOVA uses a lightweight journaling scheme that simply records the head and tail pointers for a linked list of log entries in the journal. That leads to fast garbage collection as entries are dropped from the list when they are no longer valid. There is no copying unless invalid entries make up more than half of the log, in which case a new log is created to atomically replace the old one; the metadata log entries are only copied at that point. 

He showed some performance graphs comparing the DAX versions of ext4 and XFS with NOVA. Generally, NOVA performs better than either ext4 or XFS on most [filebench](<https://github.com/filebench/filebench/wiki>) workloads that he tested. The exception is the "web server" workload where the filesystems all performed roughly the same. 

Xu said that a second RFC [posting](<https://lwn.net/Articles/749009/>) that was based on 4.16-rc4 was done in March. That post received some feedback, so he is working on those items and will be posting a v3 soon. The changes needed include 64-bit timestamps and better huge-page support. 

Chinner asked about user-space tools and, in particular, whether there was an `fsck` for NOVA. That will be needed before the filesystem can be merged as users will need to be able to repair their filesystems. Xu said there has been a focus on performance, so there is no `fsck` yet. Ted Ts'o noted that NOVA also needs a tool that can verify filesystem images, which will allow more tests in xfstests to be run on it.
