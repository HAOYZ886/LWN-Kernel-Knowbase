---
title: Asynchronous fsync()
url: https://lwn.net/Articles/789024/
date: "May 21, 2019"
category: Asynchronous IO
author: "By Jake Edge May 21, 2019 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
May 21, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The cost of [`fsync()`](<http://man7.org/linux/man-pages/man2/fsync.2.html>) is well known to filesystem developers, which is why there are [efforts](<https://lwn.net/Articles/788938/>) to provide cheaper alternatives. Ric Wheeler wanted to discuss the longstanding idea of adding an asynchronous version of `fsync()` in a filesystem session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). It turns out that what he wants may already be available via the new [io_uring](<https://lwn.net/Articles/776703/>) interface. 

The idea of an asynchronous version of `fsync()` is kind of counter-intuitive, Wheeler said. But there are use cases in large-scale data migration. If you are copying a batch of thousands of files from one server to another, you need a way to know that those files are now durable, but don't need to know that they were done in any particular order. You could find out that the data had arrived before destroying the source copies. 

It would be something like [`syncfs()`](<http://man7.org/linux/man-pages/man2/sync.2.html>) but more fine-grained so that you can select which inodes to sync, Jan Kara suggested. Wheeler said that he is not sure what the API would look like, perhaps something like `select()`. But it would be fast and useful. The idea goes back to ReiserFS, where it was discovered that syncing files in reverse order was much faster than syncing them in the order written. Ceph, Gluster, and others just need to know that all the files made it to disk in whatever order is convenient for the filesystem. 

[ ![\[Ric Wheeler\]](https://static.lwn.net/images/2019/lsf-wheeler-sm.jpg) ](<https://lwn.net/Articles/789025/>)

Chris Mason said that io_uring should be able to provide what Wheeler is looking for. He said that Jens Axboe (author of the io_uring code) already implemented an asynchronous version of [`sync_file_range()`](<http://man7.org/linux/man-pages/man2/sync_file_range.2.html>), but he wasn't sure about `fsync()`. The io_uring interface allows arbitrary operations to be done in a kernel worker thread and, when they complete, notifies user space. It would provide an [asynchronous I/O](<http://man7.org/linux/man-pages/man2/io_submit.2.html>) (AIO) version of `fsync()`, "but done properly". 

There was some discussion of io_uring and how it could be applied to various use cases. Wheeler asked if it could be used to implement what Amir Goldstein was looking for in terms of a faster `fsync()`. Mason said that he did not think so, since io_uring is restricted to POSIX operations. Goldstein agreed, saying he needed something that would not interfere with other workloads sharing the filesystem. 

Kara is concerned that an asynchronous `fsync()` as described is not really going to buy any performance gains as it will effectively become a series of `fsync()` calls on the files of interest. But Trond Myklebust said there are user-space NFS and SMB servers that might benefit from not having to tie up a thread to handle the `fsync()` calls. 

Wheeler said that if the new call just turns into a bunch of `fsync()` calls under the covers, it is not really going to help. Ts'o said that maybe what Wheeler wants is an `fsync2()` that takes an array of file descriptors and returns when they have all been synced. If the filesystem has support for `fsync2()`, it can do batching on the operations. It would be easier for application developers to call a function with an array of file descriptors rather than jumping through the hoops needed to set up an io_uring, he said. 

There is one obvious question, however: will all the files need `fsync()` or will some simply need `fdatasync()`? For a mix of operations, perhaps a flag needs to be associated with each descriptor. Kara raised the issue of file descriptors in different filesystems, though the VFS could multiplex the call to each filesystem. Wheeler wondered if it could simply be restricted to a single filesystem, but Kara said that the application may not know which filesystem the files belong to. Ts'o said it made sense to not restrict the new call to only handle files from one filesystem; it may be more of a pain for the VFS, but will be a much easier interface for application developers.
