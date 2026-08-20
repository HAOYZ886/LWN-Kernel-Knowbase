---
title: BPF iterators for filesystems
url: https://lwn.net/Articles/937326/
date: "July 6, 2023"
category: BPF; Filesystems
author: "By Jake Edge July 6, 2023 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
July 6, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

In the first of two combined BPF and filesystem sessions at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Hou Tao introduced his BPF iterators for filesystem information. [Iterators for BPF](<https://lwn.net/Articles/926041/>) are a relatively recent addition to the BPF landscape; they help BPF programs step through kernel data structures in a loop-like manner, but without running afoul of the BPF verifier, which is notoriously hard to convince about loops. 

In his remote presentation, Tao began with a quick overview of BPF iterators. They allow users to write a special type of BPF program that can step through kernel data structures in ways that would normally be handled with loops; instead, the BPF program contains callbacks that are made from the kernel in response to user-space reads of [pinned BPF files](<https://lwn.net/Articles/664688/>). The callback is made for each new kernel object encountered in the data structure; the code in the callback can then present information from the object to user space in whatever format the developer wants. 

As described in his LSFMM+BPF [topic proposal](<https://lwn.net/ml/linux-fsdevel/0a6f0513-b4b3-9349-cee5-b0ad38c81d2e@huaweicloud.com/>), Tao envisions BPF iterators being used for gathering mount and filesystem information, which was [topic of a session](<https://lwn.net/Articles/934469/>) on the previous day. The [RFC patch set](<https://lwn.net/ml/linux-fsdevel/20230507040107.3755166-1-houtao@huaweicloud.com/>) he posted a few days prior to the summit contains two iterators to extract information from specific inodes or mounts. It also has [tests to exercise the iterators](<https://lwn.net/ml/linux-fsdevel/20230507040107.3755166-5-houtao@huaweicloud.com/>). 

He then briefly described the task-file iterator, which is a BPF selftest shown in the [kernel documentation for BPF iterators](<https://docs.kernel.org/bpf/bpf_iterators.html>). A user-space program can load the BPF program containing the iterator, start the iterator with the ID of the task of interest, then use the iterator's file descriptor to read some information for the given task's open files. 

His idea is that BPF filesystem iterators would provide much more information than is currently available for various types of kernel objects, such as superblocks, inodes, directory entries (dentries), mounts, and so on. He envisions various use cases, including things like retrieving the order of the folios in the page cache, getting the page-cache information for files as an alternative to the [proposed `cachestat()` system call](<https://lwn.net/Articles/917096/>) (which was merged for 6.5), and gathering mount information in a more flexible manner than the proposed `fsinfo()` system call. 

Christian Brauner pointed out that a BPF filesystem iterator was not going to be able to replace a new system call for gathering mount information. User-space programs may not be able to—or want to—rely on BPF for getting that information. He also has some reservations about exposing mount information to BPF, in general, due to "really intricate locking scenarios". Tao thought that a BPF helper could be provided that would do the proper locking. 

The [mount iterator from the patch set](<https://lwn.net/ml/linux-fsdevel/20230507040107.3755166-5-houtao@huaweicloud.com/>) was up next. Aleksa Sarai asked if the intended users were kernel developers or regular user-space programmers; it looked to him like the iterator was meant for examining problems in the kernel. Tao agreed with that, noting that his examples were just trying to show what a BPF filesystem iterator could do. He also put up a slide showing his [inode iterator](<https://lwn.net/ml/linux-fsdevel/20230507040107.3755166-2-houtao@huaweicloud.com/>). 

After an organizer warned that time for the session was running out, Tao skipped ahead to the problems that need to be addressed. One problem is that these iterators require the `CAP_BPF` capability, so he wondered if an unprivileged BPF iterator would make sense. One way might be to allow regular users to access an iterator via a file pinned in the BPF filesystem; the permissions could be set by the administrator on the file to allow (or disallow) access. But, since the facility is targeted for debugging, it may be fine to only allow it for privileged users, he said. 

Sarai was concerned that providing that level of detail for, say, a file's layout in memory to regular users would be detrimental from a security standpoint. He thought that even if administrators could enable it for regular users, they should not do so. He was adamant that it should not be done by default; "if an admin decides to enable this, they can deal with when someone exploits it". At that point, time ran out for the session; one of the organizers suggested that the conversation continue on the mailing list.
