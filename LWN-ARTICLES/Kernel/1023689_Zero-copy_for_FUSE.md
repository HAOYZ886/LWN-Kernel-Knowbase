---
title: "Zero-copy for FUSE"
url: https://lwn.net/Articles/1023689/
date: "June 5, 2025"
category: "Filesystems-In user space; io uring"
author: "By Jake Edge June 5, 2025 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
June 5, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

In a combined storage and filesystem session at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF), Keith Busch led a discussion about zero-copy operations for the [Filesystem in Userspace](<https://github.com/libfuse/libfuse?tab=readme-ov-file#libfuse>) (FUSE) subsystem. The session was [proposed](<https://lwn.net/ml/all/dc3a5c7d-b254-48ea-9749-2c464bfd3931@davidwei.uk/>) by his colleague, David Wei, who could not make it to the summit, so Busch filled in, though he noted that ""I do not really know FUSE so well"". The idea is to eliminate data copies in the data path to and from the FUSE server in user space. 

Busch began with some background on io_uring. When an application using io_uring needs to do read and write operations on its buffers, the kernel encapsulates those buffers twice, first into an [`iov_iter`](<https://lwn.net/Articles/625077/>) (of type `ITER_UBUF`) and from that into a [`bio_vec`](<https://lwn.net/Articles/26404/>), which describes the parts of a block-I/O request. It does that for every such operation; ""if you are using the same buffer, that's kind of costly and unnecessary"". So io_uring added a way for applications to register a buffer; the kernel will create an `iov_iter` with the `ITER_BVEC` type just once when a buffer is registered. Then the application can use the io_uring "fixed" read/write operations, which will use what the kernel created rather than recreating it on each call. 

He then turned to [ublk](<https://lwn.net/Articles/903855/>), which is a block device that is implemented by a user-space server. When an application writes to the device, the ublk driver in the kernel will notify the ublk server that new data has been written to it, but the application's user-space buffer where the data lives cannot be read directly by the server. Instead, the ublk server needs to allocate a bounce buffer and ask the ublk driver to copy the data into it, which is pretty expensive. Ublk was changed in Linux 6.15 to allow the server to use the io_uring buffer registration that he had just described, so that it can do fixed read/write operations and a copy operation is not needed. 

Busch has just started looking at the FUSE code, but he thinks that the same idea could be applied for the user-space FUSE server. Now that the FUSE server (or daemon) has io_uring support, he thinks this technique could just work—the target is a file in a filesystem rather than a block device. Busch thinks that idea is different than what Wei is proposing; instead of referencing the buffers, Wei was thinking of the application sharing memory with the daemon cooperatively. Using the registration mechanism, though, would mean that the FUSE daemon would not be able to directly read the data, it would only be able to reference it for fixed io_uring operations. 

Josef Bacik agreed that Wei is looking for a way to share memory between the application and the FUSE daemon; Busch was unclear why FUSE would be needed at all in that case. Bacik said that FUSE provides files, permissions, and the like, which applications already know how to work with. The FUSE daemon may need to be able to read the data, so the registration mechanism is not sufficient. Christoph Hellwig suggested using [layout leases](<https://docs.kernel.org/admin-guide/nfs/pnfs-block-server.html>) as a way to ensure that clients have direct access to the buffer, but that the access can be revoked if the FUSE daemon exits, which Bacik and Busch thought made sense. 

Jeff Layton asked how applications would access the functionality and if a new io_uring command would be needed; Bacik thought it would just be an extension to existing io_uring commands for zero-copy networking. Stephen Bates asked if that would allow FUSE on top of a ublk device ""and that it is going to zero-copy all the way through""; Busch said that it would. 

There was some discussion of how the memory got pinned and whether it would be able to be migrated or not. In addition, there were questions about how that memory would be accounted for and, thus, would interact with memory control groups. Some of that was done without a microphone, so I was unable to fully follow, but the attendees all seemed satisfied that those concerns were being considered. 

As the session wound down, with some banter and laughter, Bates asked about what people were using ublk for. Busch said that his employer, Meta, had a [blog post](<https://engineering.fb.com/2025/03/04/data-center-engineering/a-case-for-qlc-ssds-in-the-data-center/>) about one use case which is for [quad-level cell](<https://www.purestorage.com/knowledge/what-is-qlc-flash.html>) (QLC) SSDs that are not NVMe devices. ""We are doing all the fancy stuff in user space"", so there is no out-of-tree kernel driver being used to support those devices.
