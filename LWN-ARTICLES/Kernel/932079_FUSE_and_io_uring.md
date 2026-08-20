---
title: FUSE and io_uring
url: https://lwn.net/Articles/932079/
date: "May 19, 2023"
category: "Filesystems-In user space; io uring"
author: "By Jake Edge May 19, 2023 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 19, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Bernd Schubert led a session at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>) on the [intersection](<https://lwn.net/ml/linux-fsdevel/7038cabf-e9bb-394a-e084-11bc23813fc7@ddn.com/>) of [FUSE](<https://www.kernel.org/doc/html/next/filesystems/fuse.html>) and [io_uring](<https://lwn.net/Kernel/Index/#io_uring>). He works for DDN Storage, which is using FUSE for two network-storage products; he has found FUSE to be a bottleneck for those filesystems. That could perhaps be improved by using io_uring, which is something he has been working on and wanted to discuss. 

[ ![\[Bernd Schubert\]](https://static.lwn.net/images/2023/lsfmb-schubert-sm.png) ](<https://lwn.net/Articles/932413/>)

He noted that Boaz Harrosh had [developed the zero-copy user-mode filesystem](<https://lwn.net/Articles/756625/>) (ZUFS) in 2018, but it did not go upstream in part due to [concerns about its overlap with FUSE](<https://lwn.net/Articles/787630/>). Meanwhile, Miklos Szeredi started working on FUSE2, but that has languished as well. Schubert briefly looked at both, but the [FUSE2 Git tree](<https://git.kernel.org/pub/scm/linux/kernel/git/mszeredi/fuse.git/log/?h=fuse2>) was hard to review since it is a single big patch rather than being broken into reviewable pieces. 

Last year, he was working on atomic open operations and noticed some problems; Szeredi asked for some benchmarks, which [turned out to be confusing](<https://lwn.net/ml/linux-fsdevel/20220322121212.5087-1-dharamhans87@gmail.com/>). Multiple threads were reading from `/dev/fuse`, which caused the confusing results; switching to a single thread made the results consistent. He also realized that [performing a polling loop](<https://lwn.net/ml/linux-kernel/6ba14287-336d-cdcd-0d39-680f288ca776@ddn.com/>) before adding an operation to the wait queue greatly improved the filesystem performance. 

He was also looking at an NVMe driver that his company uses, wondering why it was able to avoid the bottlenecks that he was seeing; it used io_uring, but in the "wrong" direction, from user space to the kernel, while FUSE needed to go the other way. Around that time, the [`IORING_OP_URING_CMD` support for io_uring](<https://lwn.net/Articles/844875/>) was added, which is being used in the [ublk user-space block driver](<https://lwn.net/Articles/903855/>) in the right direction, from the kernel to user space. That provides a model for doing something similar in FUSE, which is what Schubert has been working on. 

He explained the inner workings of the work he has done to make FUSE use io_uring. There is one thread per core, each with its own ring buffer; there is a shared memory buffer with the FUSE queue ID used as the offset for user space to `mmap()` its region. Libfuse initiates operations with an `IORING_OP_URING_CMD`, which is the core idea taken from ublk. For debugging purposes, there is a mode with a single thread and ring buffer. 

Amir Goldstein asked whether user space really needed to be aware of the underlying implementation. Schubert replied that his goal was to make it transparent, so that existing filesystem implementations could gain the performance benefits without having to change their code. 

He is unhappy with using the queue ID to identify the user-space buffer (via the `mmap()` offset), but was unable to find the corresponding buffer in the kernel without it. There was some discussion of ways to get the kernel and user-space in sync on the buffer location directly. Jan Kara suggested looking in the VMA associated with the user-space virtual address to find where in kernel memory the buffer was located; he said that he could help Schubert find the right calls to make for that. 

Schubert showed some performance benchmarks, but noted that he needs to find a way to keep the scheduler from migrating the application processes to other CPUs. For example, direct I/O reads showed moderate improvements for io_uring over regular FUSE, but much larger improvements when migration was disabled. Kara cautioned that CPU migration can be a problem at the start of a test like this, but may not actually be problematic over the long term; meanwhile, there are other workloads that may benefit from the migration. 

But Schubert said that FUSE is particularly affected by this because most of the I/O work is being handed off to another process; once it completes, it is best if the application process is still running on that same CPU. Kara said that it is not a simple problem and that the scheduler may lack the information it needs to make the right decision. The scheduler developers are aware that there may be problems in scheduling for io_uring and are working on some solutions; he recommended that Schubert work with those developers.
