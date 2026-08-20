---
title: A new kernel polling interface
url: https://lwn.net/Articles/743714/
date: "January 9, 2018"
category: Asynchronous IO; poll
author: "By Jonathan Corbet January 9, 2018"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 9, 2018

Polling a set of file descriptors to see which ones can perform I/O without blocking is a useful thing to do — so useful that the kernel provides three different system calls ([`select()`](<http://man7.org/linux/man-pages/man2/select.2.html>), [`poll()`](<http://man7.org/linux/man-pages/man2/poll.2.html>), and [`epoll_wait()`](<http://man7.org/linux/man-pages/man2/epoll_wait.2.html>) — plus some variants) to perform it. But sometimes three is not enough; there is now a proposal circulating for a fourth kernel polling interface. As is usually the case, the motivation for this change is performance. 

On January 4, Christoph Hellwig posted [a new polling API](<https://lwn.net/Articles/742978/>) based on the asynchronous I/O (AIO) mechanism. This may come as a surprise to some, since AIO is not the most loved of kernel interfaces and it tends not to get a lot of attention. AIO allows for the submission of I/O operations without waiting for their completion; that waiting can be done at some other time if need be. The kernel has had AIO support since the 2.5 days, but it has always been somewhat incomplete. Direct file I/O (the original use case) works well, as does network I/O. Many other types of I/O are not supported for asynchronous use, though; attempts to use the AIO interface with them will yield synchronous behavior. In a sense, polling is a natural addition to AIO; the whole point of polling is usually to avoid waiting for operations to complete. 

The patches add a new command (`IOCB_CMD_POLL`) that can be passed in an I/O control block (IOCB) to [`io_submit()`](<http://man7.org/linux/man-pages/man2/io_submit.2.html>) along with any of the usual `POLL*` flags describing the type of I/O that is desired — `POLLIN` for data available to read, for example. This command, like other AIO commands, will not (necessarily) complete before `io_submit()` returns. Instead, when the indicated file descriptor is ready for the requested type of I/O, a completion event will be queued. A subsequent call to [`io_getevents()`](<http://man7.org/linux/man-pages/man2/io_getevents.2.html>) (or the `io_pgetevents()` variant, added by the patch set, that blocks signals during the operation) will return that event, and the calling application will know that it can perform I/O on the indicated file descriptor. AIO poll operations always operate in the "one-shot" mode; once a poll notification has been generated, a new `IOCB_CMD_POLL` IOCB must be submitted for that file descriptor if further notifications are needed. 

Thus far, this interface sounds more difficult to use than the existing poll system calls. There is a payoff, though, that comes in the form of the AIO ring buffer. This poorly documented aspect of the AIO subsystem maps a circular buffer into the calling process's address space. That process can then consume notification events directly from the buffer rather than calling `io_getevents()`. Multiple notifications can be consumed without the need to enter the kernel at all, and polling for multiple file descriptors can be re-established with a single `io_submit()` call. The result, Hellwig said in the patch posting, is an up-to-10% improvement in the performance of the [Seastar](<http://www.seastar-project.org/>) I/O framework. More recently, he [noted](<https://lwn.net/Articles/743723/>) that the improvement grows to 16% on kernels with [page-table isolation](<https://lwn.net/Articles/741878/>) turned on. 

Internally to the kernel, any device driver (or other subsystem that exports a [`file_operations`](<http://elixir.free-electrons.com/linux/v4.14.12/source/include/linux/fs.h#L1692>) structure) can support the new poll interface, but some small changes will be required. It is not, however, necessary to support (or even know about) AIO in general. In current kernels, the polling system calls are all supported by the `poll()` method in `struct file_operations`: 

```
int (*poll) (struct file *file, struct poll_table_struct *table);
```

This function must perform two actions: setting up notifications for when the underlying `file` is ready for I/O, and returning the types of I/O that could be performed without blocking now. The first is done by adding one or more wait queues to the provided `table`; the driver will perform a wakeup call on one of those queues when the state of the device changes. The current readiness state is the return value from the `poll()` method itself. 

Supporting AIO-based polling requires splitting those two functions into separate `file_operations` methods. Thus, there are two new entries to that structure: 

```
struct wait_queue_head *(*get_poll_head)(struct file *file, int mask);
        int (*poll_mask) (struct file *file, int mask);
```

(The actual patches use the new typedef `__poll_t` for the `mask`, but that typedef isn't in the mainline kernel yet). The polling subsystem will call `get_poll_head()` to obtain a pointer to the wait queue that will be notified when the device's I/O readiness state changes; `poll_mask()` will be called to get the current readiness state. A driver that implements these two operations need not (and probably should not) retain its implementation of the older `poll()` interface. 

One potential limitation built into this API is that there can only be a single wait queue that receives notifications for a given `file`. The current interface, instead, allows multiple queues to be used, and a number of drivers take advantage of that fact to use, for example, different queues for read and write readiness. Contemporary wait queues offer enough flexibility that the use of multiple queues should not be necessary anymore. If a driver cannot be changed, Hellwig said, ""the driver just won't support aio poll"" 

There have not been a lot of comments in response to the patch posting so far; many of the relevant developers have been preoccupied with other issues in the last week. It is hard to argue with a 10% performance improvement, though, so some form of this patch seems likely to get into the mainline sooner or later — interested parties can keep checking the mainline repository to see if it's there yet. Whether we'll see a fifth polling interface added in the future is anybody's guess, though.
