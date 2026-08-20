---
title: "Two new system calls: splice() and sync_file_range()"
url: https://lwn.net/Articles/178199/
date: "April 3, 2006"
category: posix fadvise; splice; sync file range
author: 
---

The 2.6.17 kernel will include two new system calls which expand the capabilities available to user-space programs in some interesting ways. This article contains a look at the current form of these new interfaces. 

### splice()

The `splice()` system call has a long history. First proposed by Larry McVoy in 1998; it was seen as a way of improving I/O performance on server systems. Despite being often mentioned in the following years, no `splice()` implementation was ever created for the mainline Linux kernel. That situation changed, however, just before the 2.6.17 merge window was closed when Jens Axboe's `splice()` patch, along with a number of modifications, was merged. 

As of this writing, the `splice()` interface looks like this: 

```
long splice(int fdin, int fdout, size_t len, unsigned int flags);
```

A call to `splice()` will cause the kernel to move up to `len` bytes from the data source `fdin` to `fdout`. The data will move through kernel space only, with a minimum of copying. In the current implementation, at least one of the two file descriptors must refer to a pipe device. That requirement is a limitation of the current code, and it could be removed at some future time. 

The `flags` argument modifies how the copy is done. Currently implemented flags are: 

  * `SPLICE_F_NONBLOCK`: makes the `splice()` operations non-blocking. A call to `splice()` could still block, however, especially if either of the file descriptors has not been set for non-blocking I/O. 

  * `SPLICE_F_MORE`: a hint to the kernel that more data will come in a subsequent `splice()` call. 

  * `SPLICE_F_MOVE`: if the output is a file, this flag will cause the kernel to attempt to move pages directly from the input pipe buffer into the output address space, avoiding a copy operation. 

Internally, `splice()` works using the [pipe buffer](<http://lwn.net/Articles/119682/>) mechanism added by Linus in early 2005 - that is why one side of the operation is required to be a pipe for now. There are two additions to the ever-growing `file_operations` structure for devices and filesystems which wish to support `splice()`: 

```
ssize_t (*splice_write)(struct inode *pipe, struct file *out, 
                                size_t len, unsigned int flags);
        ssize_t (*splice_read)(struct file *in, struct inode *pipe, 
                               size_t len, unsigned int flags);
```

The new operations should move `len` bytes between `pipe` and either `in` or `out`, respecting the given `flags`. For filesystems, there are generic implementations of these operations which can be used; there is also a `generic_splice_sendpage()` which is used to enable splicing to a socket. As of this writing, there are no `splice()` implementations for device drivers, but there is nothing preventing such implementations in the future, for char devices at least. 

Discussions on the linux-kernel suggest that the `splice()` interface could change before it is set in stone with the 2.6.17 release. Andrew Tridgell has [requested](<https://lwn.net/Articles/178208/>) that an offset argument be added to specify where copying should begin - either that, or a separate `psplice()` should be added. There is also some concern about error handling; if a `splice()` call returns an error, how does the application tell whether the problem is with the input or the output? Resolving those issues may require some interface changes over the next month or so. 

### sync_file_range()

Early in the 2.6.17 process, some [changes to the `posix_fadvise()` system call](<http://lwn.net/Articles/171941/>) were merged. The new, Linux-specific options were meant to give applications better control over how data written to files is flushed to the physical media. The capabilities provided are needed, but there were concerns about extending a POSIX-defined function in a Linux-specific way. So, after some discussions, Andrew Morton pulled that patch back out and replaced it with a new system call: 

```
long sync_file_range(int fd, loff_t offset, loff_t nbytes, int flags);
```

This call will synchronize a file's data to disk, starting at the given `offset` and proceeding for `nbytes` bytes (or to the end of the file if `nbytes` is zero). How the synchronization is done is controlled by `flags`: 

  * `SYNC_FILE_RANGE_WAIT_BEFORE` blocks the calling process until any already in-progress writeout of pages (in the given range) completes. 

  * `SYNC_FILE_RANGE_WRITE` starts writeout of any dirty pages in the given range which are not already under I/O. 

  * `SYNC_FILE_RANGE_WAIT_AFTER` blocks the calling process until the newly-initiated writes complete. 

An application which wants to initiate writeback of all dirty pages should provide the first two flags. Providing all three flags guarantees that those pages are actually on disk when the call returns. 

The new implementation avoids distorting the `posix_fadvise()` system call. It also allows synchronization operations to be performed with a single call, instead of the multiple calls required by the previous attempt. In the future, it may also be possible to add other operations to the `flags` list; the ability to request metadata synchronization seems to be high on the list. 

(Thanks to Michael Kerrisk - who agitated for this change - for providing some of the background information).
