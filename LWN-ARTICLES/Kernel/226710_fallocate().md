---
title: fallocate()
url: https://lwn.net/Articles/226710/
date: "March 19, 2007"
category: fallocate
author: 
---

Applications do not normally worry about the allocation of blocks for files they create; instead, they simply write the data and assume the the kernel will do a proper job of finding a home for that data. There are times when it is useful to take a more active role in block allocation, though. If an application knows how much data it will be writing, it can request the needed blocks ahead of time, enabling the kernel to allocate them all at once, contiguously on the disk. Application developers concerned about reliability may also want to know that the needed disk space has already been procured before beginning a critical operation. 

Unix systems have not traditionally provided a way for applications to control block allocation. An application on a current Linux kernel has only one way to force allocation: write a stream of data to the relevant portion of the file. This technique works, but it loses one of the advantages of preallocation: letting the kernel do all the work at once and ensure that the blocks are contiguous on disk if possible. Writing useless data to the disk solely for the purpose of forcing block allocation is also wasteful. 

The POSIX way of preallocating disk space is the [`posix_fallocate()`](<http://www.opengroup.org/onlinepubs/000095399/functions/posix_fallocate.html>) system call, defined as: 

```
int posix_fallocate(int fd, off_t offset, off_t len);
```

On success, this call will ensure that the application can write up to `len` bytes to `fd` starting at the given `offset` and know that the disk space is there for it. 

Linux does not currently have an implementation of `posix_fallocate()` in the kernel. [This patch by Amit Arora](<http://lwn.net/Articles/226436/>) may change that situation, however. Amit's patch has been through a couple of rounds of review which have changed the interface considerably; the current form of the proposed system call is: 

```
long fallocate(int fd, int mode, loff_t offset, loff_t len);
```

The `fd`, `offset`, and `len` arguments have the same meaning as with `posix_fallocate()`, making it easy for the C library to implement the standard interface. The additional `mode` argument changes the way the call operates; normal usage will be to specify `FA_ALLOCATE`, which causes the requested blocks to be allocated. If, instead, `FA_DEALLOCATE` is given, the requested block range will be deallocated, allowing an application to punch a hole in the file. 

Internally, the system call does not do much of the work; instead, it calls the new `fallocate()` inode operation. Thus, each filesystem must implement its own `fallocate()` support. The future plans call for a possible generic implementation for filesystems which lack `fallocate()` support, but the generic version would almost certainly have to rely on writing zeroes to the file. By pushing the operation into the filesystem itself, the kernel gives the filesystem the opportunity to satisfy the allocation in a more efficient way, without the need to write filler data. Filesystems do need to be sure that applications cannot use `fallocate()` to read old data from the allocated blocks, though. 

For now, filesystem-level support is scarce. There are patches circulating which add `fallocate()` support to ext4. The XFS filesystem has supported preallocation (through a special `ioctl()` call) for some time, but will need to be modified to do preallocation through the new inode operation. It's not clear when other filesystems may get native support; the tracking of allocated but unwritten blocks is a significant addition. So, for the near future, the efficiency benefits of `fallocate()` may be unavailable for most users.
