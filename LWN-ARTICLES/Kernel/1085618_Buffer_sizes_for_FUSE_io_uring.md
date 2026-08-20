---
title: Buffer sizes for FUSE io_uring
url: https://lwn.net/Articles/1085618/
date: "August 3, 2026"
category: "Filesystems-In user space"
author: "By Jake Edge August 3, 2026 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
August 3, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

The [Filesystem in Userspace](<https://docs.kernel.org/filesystems/fuse/>) (FUSE) subsystem provides a way to service filesystem requests from a user-space server, which moves the format-handling code out of the kernel. The FUSE server can [use](<https://www.kernel.org/doc/html/next/filesystems/fuse-io-uring.html>) the [io_uring](<https://man7.org/linux/man-pages/man7/io_uring.7.html>) facility for better performance, but Bernd Schubert is concerned that memory is being wasted because the current implementation has a single, large buffer size that is excessive for small I/O operations. He led a discussion on that topic in the filesystem track of the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) in Zagreb, Croatia. 

Currently, [libfuse](<https://github.com/libfuse/libfuse#libfuse>) sets up eight entries per ring, with each entry defaulting to the maximum payload size (8MB), though switching to 4MB is also possible, he began. Normally one or two entries per ring are needed to achieve maximum throughput to the disk (which can be local or over the network), but there is also a need to handle metadata requests, for operations like [`stat()`](<https://man7.org/linux/man-pages/man2/stat.2.html>) or directory listings. ""You don't want to disturb your streaming I/O while you are doing these metadata requests"". 

Those smaller requests might be 16KB or 128KB. In order to saturate the I/O bandwidth with requests of that size, however, a larger number of buffers is needed. ""The question is how we do this."" The existing implementation uses entries with a separate header and payload structure, but each has the same payload size. 

Schubert noted that Joanne Koong has been working on [patches to change the buffer handling for FUSE io_uring](<https://lwn.net/ml/all/20260402162840.2989717-1-joannelkoong@gmail.com/>) that would separate the headers from the payloads. Each entry would still have a header, but the payload buffers would be shared. There would no longer be a one-to-one mapping between entries and payloads, and the payloads would be available in multiple sizes. 

[ ![\[Bernd Schubert\]](https://static.lwn.net/images/2026/lsfmb-schubert-sm.png) ](<https://lwn.net/Articles/1086043/>)

Schubert has envisioned a different approach with, say, four different payload sizes. There would be entries that used 4KB, 64KB, 128KB, and 1MB payload sizes; the 4KB entries would have 128 buffers available, while the larger sizes would only have two buffers available. He has not written the code to implement his idea, but he thinks that it would work: the right-sized entry from a single ring would be chosen based on the size of the I/O operation. 

In a discussion that he had with Koong the day before, she said that the problem could be solved with multiple rings instead of multiple sizes on a single ring. Each ring would have a different buffer size and there would be code to choose the right one based on the size of the I/O, which is similar to the code that Schubert envisions. His concern is that having multiple rings means making more system calls, so his preference would be to have a single ring. The user-space FUSE server would need to monitor multiple rings and service I/O operations from each. 

Another concern that he has with multiple rings is for applications on systems partitioned to run Linux on just a few cores, which means that a single core may be responsible for polling multiple rings. ""That gets painful."" 

FUSE maintainer Miklos Szeredi asked whether Schubert's plan required managing buffer allocations in a large pool of memory, but Schubert said that it did not. Handling allocations that way would lead to fragmentation, so his method simply statically sets up multiple buffers of each size; when there are no more 4KB buffers, for example, the requester will have to wait. There was some question about perhaps using the larger sizes in that case, but he thinks those are too precious to be used that way. 

Koong said that she is concerned that waiting for an available buffer of the right size will add unneeded latency. With her multi-ring solution, the other rings can provide a buffer for a request instead of waiting for one of the right size. She asked: ""If you have a small request and there are larger buffers available, why are you making the small request wait until there is a small buffer available?"" Schubert said that in that case, there are already 128 requests in flight, so adding another has no real benefit. 

There was some back and forth between them, with Schubert wanting to avoid the user-space complexity of handling multiple io_uring rings, while Koong is looking to make the best use of all available buffers. She said that she did not see a need for more than two different buffer sizes, one small for metadata and one large for data. Schubert said that was fine, but that it still meant polling multiple rings if the different sizes are on different rings as Koong has proposed. 

An attendee asked about the io_uring performance problems that are being seen, but Schubert said that it is not performance that he is trying to improve. He wants to reduce the memory usage as there is a need for many metadata requests, but if they all have to be the same size as the I/O requests, lots of memory is wasted. 

Koong raised the issue of [head-of-line blocking](<https://en.wikipedia.org/wiki/Head-of-line_blocking>), which she said was an advantage to having multiple rings: one thread could be handling the data I/O while another was handling the metadata requests. Schubert said that the current libfuse may still be handling requests in a synchronous fashion, which could cause smaller I/O operations to block behind the larger ones, but that switching libfuse to handle io_uring operations asynchronously should solve that problem. Koong was not convinced that a libfuse change of that sort would work well with custom libraries that have their own idea of how the I/O will be handled. Schubert agreed that could be a problem, but thought it was rare, which Koong disagreed with. 

Schubert said that he was planning to add coroutine support to libfuse, which should make it possible for servers to avoid the head-of-line blocking problem when only using the libfuse thread. He noted that it is already possible to write a FUSE server that avoids the problem, but it must do so using additional threads that handle asynchronous io_uring operations. 

At that point, the session had run out of time and it was not entirely clear where things go from there.
