---
title: The rapid growth of io_uring
url: https://lwn.net/Articles/810414/
date: "January 24, 2020"
category: Asynchronous IO; io uring
author: "By Jonathan Corbet January 24, 2020"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 24, 2020

One year ago, the [io_uring subsystem](<https://lwn.net/Articles/776703/>) did not exist in the mainline kernel; it showed up in the 5.1 release in May 2019\. At its core, io_uring is a mechanism for performing asynchronous I/O, but it has been steadily growing beyond that use case and adding new capabilities. Herein we catch up with the current state of io_uring, where it is headed, and an interesting question or two that will come up along the way. 

Classic Unix I/O is inherently synchronous. As far as an application is concerned, an operation is complete once a system call like `read()` or `write()` returns, even if some processing may continue behind its back. There is no way to launch an operation asynchronously and wait for its completion at some future time — a feature that many other operating systems had for many years before Unix was created. 

In the Linux world, this gap was eventually filled with the [asynchronous I/O (AIO) subsystem](<http://man7.org/linux/man-pages/man2/io_setup.2.html>), but that solution has never proved to be entirely satisfactory. AIO requires specific support at the lower levels, so it never worked well outside of a couple of core use cases (direct file I/O and networking). Over the years there have been recurring conversations about better ways to solve the asynchronous-I/O problem. Various proposals with names like [fibrils](<https://lwn.net/Articles/219954/>), [threadlets](<https://lwn.net/Articles/223899/>), [syslets](<https://lwn.net/Articles/236206/>), [acall](<https://lwn.net/Articles/316806/>), and [work-queue-based AIO](<https://lwn.net/Articles/671649/>) have been discussed, but none have made it into the mainline. 

The latest attempt in that series is io_uring, which _did_ manage to get merged. Unlike its predecessors, io_uring is built around a ring buffer in memory shared between user space and the kernel; that allows the submission of operations (and collecting the results) without the need to call into the kernel in many cases. The interface is somewhat complex, but for many applications that perform massive amounts of I/O, that complexity is paid back in increased performance. See [this document [PDF]](<https://kernel.dk/io_uring.pdf>) for a detailed description of the io_uring API. Use of this API can be somewhat simplified with the [liburing library](<https://github.com/axboe/liburing>). 

#### What io_uring can do

Every entry placed into the io_uring submission ring carries an opcode telling the kernel what is to be done. When io_uring was added to the 5.1 kernel, the available opcodes were: 

> `IORING_OP_NOP`
>     This operation does nothing at all; the benefits of doing nothing asynchronously are minimal, but sometimes a placeholder is useful. 
> 
> `IORING_OP_READV`
> `IORING_OP_WRITEV`
>     Submit a [`readv()` or `write()`](<http://man7.org/linux/man-pages/man2/readv.2.html>) operation — the core purpose for io_uring in most settings. 
> 
> `IORING_OP_READ_FIXED`
> `IORING_OP_WRITE_FIXED`
>     These opcodes also submit I/O operations, but they use "registered" buffers that are already mapped into the kernel, reducing the amount of total overhead. 
> 
> `IORING_OP_FSYNC`
>     Issue an [`fsync()`](<http://man7.org/linux/man-pages/man2/fsync.2.html>) call — asynchronous synchronization, in other words. 
> 
> `IORING_OP_POLL_ADD`
> `IORING_OP_POLL_REMOVE`
>     `IORING_OP_POLL_ADD` will perform a [`poll()`](<http://man7.org/linux/man-pages/man2/poll.2.html>) operation on a set of file descriptors. It's a one-shot operation that must be resubmitted after it completes; it can be explicitly canceled with `IORING_OP_POLL_REMOVE`. Polling this way can be used to asynchronously keep an eye on a set of file descriptors. The io_uring subsystem also supports a concept of dependencies between operations; a poll could be used to hold off on issuing another operation until the underlying file descriptor is ready for it. 

That functionality was enough to drive some significant interest in io_uring; its creator, Jens Axboe, could have stopped there and taken a break for a while. That, however, is not what happened. Since the 5.1 release, the following operations have been added: 

> `IORING_OP_SYNC_FILE_RANGE` (5.2)
>     Perform a [`sync_file_range()`](<http://man7.org/linux/man-pages/man2/sync_file_range.2.html>) call — essentially an enhancement of the existing `fsync()` support, though without all of the guarantees of `fsync()`. 
> 
> `IORING_OP_SENDMSG` (5.3)
> `IORING_OP_RECVMSG` (5.3)
>     These operations support the asynchronous sending and receiving of packets over the network with [`sendmsg()`](<http://man7.org/linux/man-pages/man3/sendmsg.3p.html>) and [`recvmsg()`](<http://man7.org/linux/man-pages/man3/recvmsg.3p.html>). 
> 
> `IORING_OP_TIMEOUT` (5.4)
> `IORING_OP_TIMEOUT_REMOVE` (5.5)
>     This operation completes after a given period of time, as measured either in seconds or number of completed io_uring operations. It is a way of forcing a waiting application to wake up even if it would otherwise continue sleeping for more completions. 
> 
> `IORING_OP_ACCEPT` (5.5)
> `IORING_OP_CONNECT` (5.5)
>     Accept a connection on a socket, or initiate a connection to a remote peer.
> 
> `IORING_OP_ASYNC_CANCEL` (5.5)
>     Attempt to cancel an operation that is currently in flight. Whether this attempt will succeed depends on the type of operation and how far along it is. 
> 
> `IORING_OP_LINK_TIMEOUT` (5.5)
>     Create a timeout linked to a specific operation in the ring. Should that operation still be outstanding when the timeout happens, the kernel will attempt to cancel the operation. If, instead, the operation completes first, the timeout will be canceled. 

That is where the io_uring interface will stand as of the final 5.5 kernel release. 

#### Coming soon

The development of io_uring is far from complete. To see that, one need merely look into linux-next to see what is queued for 5.6: 

> `IORING_OP_FALLOCATE`
>     Manipulate the blocks allocated for a file using [`fallocate()`](<http://man7.org/linux/man-pages/man2/fallocate.2.html>)
> 
> `IORING_OP_OPENAT`
> `IORING_OP_OPENAT2`
> `IORING_OP_CLOSE`
>     Open and close files
> 
> `IORING_OP_FILES_UPDATE`
>     Frequently used files can be registered with io_uring for faster access; this command is a way of (asynchronously) adding files to the list (or removing them from the list). 
> 
> `IORING_OP_STATX`
>     Query information about a file using [`statx()`](<http://man7.org/linux/man-pages/man2/statx.2.html>). 
> 
> `IORING_OP_READ`
> `IORING_OP_WRITE`
>     These are like `IORING_OP_READV` and `IORING_OP_WRITEV`, but they use the simpler interface that can only handle a single buffer. 
> 
> `IORING_OP_FADVISE`
> `IORING_OP_MADVISE`
>     Perform the [`posix_fadvise()`](<http://man7.org/linux/man-pages/man2/posix_fadvise.2.html>) and [`madvise()`](<http://man7.org/linux/man-pages/man2/madvise.2.html>) system calls asynchronously. 
> 
> `IORING_OP_SEND`
> `IORING_OP_RECV`
>     Send and receive network data.
> 
> `IORING_OP_EPOLL_CTL`
>     Perform operations on epoll file-descriptor sets with [`epoll_ctl()`](<http://man7.org/linux/man-pages/man2/epoll_ctl.2.html>)

What will happen after 5.6 remains to be seen. There was [an attempt](<https://lwn.net/ml/io-uring/f77ac379ddb6a67c3ac6a9dc54430142ead07c6f.1576336565.git.asml.silence@gmail.com/>) to add [`ioctl()`](<http://man7.org/linux/man-pages/man2/ioctl.2.html>) support, but that was [shot down](<https://lwn.net/ml/io-uring/CAG48ez0N_b+kjbddhHe+BUvSnOSvpm1vdfQ9cv+cgTLuCMXqug@mail.gmail.com/>) due to reliability and security concerns. Axboe has, however, [outlined](<https://lwn.net/ml/io-uring/1f995281-4a56-a7de-d20b-14b0f64536c0@kernel.dk/>) a way in which support for specific `ioctl()` operations could be added on a case-by-case basis. One can imagine that, for example, the media subsystem, which supports a number of performance-sensitive `ioctl()` operations, would benefit from this mechanism. 

There is also [an early patch set](<https://lwn.net/ml/io-uring/cover.1579649589.git.asml.silence@gmail.com/>) adding support for [`splice()`](<http://man7.org/linux/man-pages/man2/splice.2.html>). 

#### An asynchronous world

All told, it would appear that io_uring is quickly growing the sort of capabilities that were envisioned many years ago when the developers were talking about thread-based asynchronous mechanisms. The desire to avoid blocking in event loops is strong; it seems likely that this API will continue to grow until a wide range of tasks can be performed with almost no risk of blocking at all. Along the way, though, there may be a couple of interesting issues to deal with. 

One of those is that the field for io_uring commands is only eight bits wide, meaning that up to 256 opcodes can be defined. As of 5.6, 30 opcodes will exist, so there is still plenty of room for growth. There are more than 256 system calls implemented in Linux, though. If io_uring were to grow to the point where it supported most of them, that space would run out. 

A different issue was [raised](<https://lwn.net/ml/linux-fsdevel/d4d3fa40-1c59-a48a-533b-c8b221e0f221@samba.org/>) by Stefan Metzmacher. Dependencies between commands are supported by io_uring now, so it is possible to hold the initiation of an operation until some previous operation has completed. What is rather more difficult is moving information between operations. In Metzmacher's case, he would like to call [`openat()`](<http://man7.org/linux/man-pages/man2/open.2.html>) asynchronously, then submit I/O operations on the resulting file descriptor without waiting for the open to complete. 

It [turns out](<https://lwn.net/ml/linux-fsdevel/7324bbb7-8f7b-c0c6-6a45-48b8b77c4be8@kernel.dk/>) that there is a plan for this: inevitably it calls for ... wait for it ... using BPF to make the connection from one operation to the next. The ability to run bits of code in the kernel at appropriate places in a chain of asynchronous operations would clearly open up a number of interesting new possibilities. ""There's a lot of potential there"", Axboe said. Indeed, one can imagine a point where an entire program is placed into a ring by a small C "driver", then mostly allowed to run on its own. 

There is one potential hitch here, though, in that io_uring is an unprivileged interface; any necessary privilege checks are performed on the actual operations performed. But the plans to make BPF safe for unprivileged users [have been sidelined](<https://lwn.net/Articles/796328/>), with explicit statements that unprivileged use will not be supported in the future. That could make BPF hard to use with io_uring. There may be plans for how to resolve this issue lurking deep within Facebook, but they have not yet found their way onto the public lists. It appears that the BPF topic in general [will be discussed](<https://lwn.net/ml/linux-fsdevel/e25f7a09-96b2-2288-4777-9f728a8b2c23@gmail.com/>) at the 2020 Linux Storage, Filesystem, and Memory-Management Summit. 

In summary, though, io_uring appears to be on a roll with only a relatively small set of growing pains. It will be interesting to see how much more functionality finds its way into this subsystem in the coming releases. Recent history suggests that the growth of io_uring will not be slowing down anytime soon.
