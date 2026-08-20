---
title: Automatic buffer selection for io_uring
url: https://lwn.net/Articles/815491/
date: "March 20, 2020"
category: Asynchronous IO; io uring
author: "By Jonathan Corbet March 20, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 20, 2020

The [io_uring subsystem](<https://lwn.net/Articles/776703/>) has, in the last year, redefined how asynchronous I/O is done on Linux systems. As this subsystem grows in both capability and users, though, it starts to run into limitations in the types of operations that can be expressed. That is driving a number of changes in how operations are programmed for io_uring. One example is the mechanisms considered for carrying a file descriptor between operations that was [covered here](<https://lwn.net/Articles/813827/>) in early March. Another has to do with how I/O buffers are chosen for operations. 

As io_uring developer Jens Axboe describes in [this patch set](<https://lwn.net/ml/io-uring/20200310150427.28489-1-axboe@kernel.dk/>), the usual mode for programs that handle large numbers of file descriptors is to use [`poll()`](<http://man7.org/linux/man-pages/man2/poll.2.html>) to find out which descriptors are ready for I/O, then making separate calls to actually perform that I/O. One could use io_uring in this mode, but it defeats one of the purposes of the whole exercise: avoiding system calls whenever possible. The io_uring way of doing things is to just queue an asynchronous operation on every file descriptor, then react to the resulting events whenever one of those operations is executed. 

Working that way can indeed reduce system calls — all the way to zero if the request ring is kept full. But it also requires allocating a separate I/O buffer for each of those queued operations, even though many of them may not execute for an indefinite period of time. The `poll()` method, instead, allows an application to defer buffer allocation until a buffer is actually needed. Losing that flexibility can result in significantly higher memory use for applications that keep a large number of operations outstanding. 

What is needed here is some sort of mechanism that allows buffers to be allocated to operations after they have been queued in the ring. The answer is, of course, obvious: add a hook for a BPF program that can perform buffer management in the kernel at the moment that any particular operation is able to go forward. Why even try anything else? Unfortunately, Axboe said, ""I had a hard time imagining how life times of the buffer could be managed through [BPF]"", so that idea went by the wayside. 

Fortunately, there was another idea waiting in the wings: have the application provide one or more buffer pools to io_uring, which would then select a buffer from one of those pools whenever one is needed. That is what Axboe ended up implementing. 

To use this mechanism, an application starts by queuing one or more `IORING_OP_PROVIDE_BUFFERS` operations to give a set of I/O buffers to the kernel. Each operation includes the base address of the buffer(s), a count of buffers, the size (the same for all buffers in this operation), a base buffer ID, and a group ID. If more than one buffer is included in the request, the buffer ID will be incremented by one for each after the first. There is no requirement that all buffers in a given group be the same size, but that seems to be the way that the mechanism is intended to be used. 

Subsequently, operations can be queued without providing buffers at submission time; instead, the special value `IOSQE_BUFFER_SELECT` is used. The new `buf_group` field in the queue entry should be set to the ID of the group from which a buffer should be obtained when needed. When an operation unblocks and can proceed, the kernel will grab a buffer from the indicated group and use it. The size of the buffer is not considered during the selection process so, if the buffer is too small, the operation will not be able to complete properly. The ID of the selected buffer is returned with the operation's completion status. 

If the requested buffer group is empty, the operation will fail with an `ENOBUFS` error. Once a buffer has been consumed by an operation, the kernel will not use it again until it has been given back with another `IORING_OP_PROVIDE_BUFFERS` request. 

Only some operations support buffer selection in the current patch set; it is limited to `read()`, `readv()`, `recv()`, and `recvmsg()`. Earlier versions of the patch set supported `write()`, though your editor will freely admit to being baffled with regard to how that was supposed to actually work even after looking at the code; that support was removed in [version 3](<https://lwn.net/ml/io-uring/20200228203053.25023-1-axboe@kernel.dk/>). 

This work has not yet found its way into linux-next, but there is still some time before the 5.7 merge window opens. So there is a chance that the buffer-selection feature could yet land in the next development cycle. That will increase the flexibility of io_uring operations, and no BPF hooks are required.
