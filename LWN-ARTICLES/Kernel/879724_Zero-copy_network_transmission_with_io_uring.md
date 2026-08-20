---
title: "Zero-copy network transmission with io_uring"
url: https://lwn.net/Articles/879724/
date: "December 30, 2021"
category: "Asynchronous IO; Networking-Performance; io uring"
author: "By Jonathan Corbet December 30, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 30, 2021

When the goal is to push bits over the network as fast as the hardware can go, any overhead hurts. The cost of copying data to be transmitted from user space into the kernel can be especially painful; it adds latency, takes valuable CPU time, and can be hard on cache performance. So it is unsurprising that the developers working with [io_uring](<https://lwn.net/Articles/776703/>), which is all about performance, have turned their attention to zero-copy network transmission. [This patch set](<https://lwn.net/ml/linux-kernel/cover.1640029579.git.asml.silence@gmail.com/>) from Pavel Begunkov, now in its second revision, looks to be significantly faster than the [`MSG_ZEROCOPY` option](<https://lwn.net/Articles/726917/>) supported by current kernels. 

As a reminder: io_uring is a relatively new API for asynchronous I/O (and related operations); it was first merged less than three years ago. User space sets up a pair of circular buffers shared with the kernel; the first buffer is used to submit operations to the kernel, while the second receives the results when operations complete. A suitably busy process that keeps the submission ring full can perform an indefinite number of operations without needing to make any system calls, which clearly improves performance. Io_uring also implements the concept of "fixed" buffers and files; these are held open, mapped, and ready for I/O within the kernel, saving the setup and teardown overhead that is otherwise incurred by every operation. It all adds up to a significantly faster way for I/O-intensive applications to work. 

One thing that io_uring still does not have is zero-copy networking, even though the networking subsystem supports zero-copy operation via the `MSG_ZEROCOPY` socket option. In theory, adding that support is simply a matter of wiring up the integration between the two subsystems. In practice, naturally, there are a few more details to deal with. 

A zero-copy networking implementation must have a way to inform applications when any given operation is truly complete; the application cannot reuse a buffer containing data to be transmitted if the kernel is still working on it. There is a subtle point that is relevant here: the completion of a [`send()`](<https://man7.org/linux/man-pages/man2/send.2.html>) call (for example) does not imply that the associated buffer is no longer in use. The operation "completes" when the data has been accepted into the networking subsystem for transmission; the higher layers may well be done with it, but the buffer itself may still be sitting in a network interface's transmission queue. A zero-copy operation is only truly done with its data buffers when the hardware has done its work — and, for many protocols, when the remote peer has acknowledged receipt of the data. That can happen long after the operation that initiated the transfer has completed. 

So there needs to be a mechanism by which the kernel can tell applications that a given buffer can be reused. `MSG_ZEROCOPY` handles this by returning notifications via the error queue associated with the socket — a bit awkward, but it works. Io_uring, instead, already has a completion-notification mechanism in place, so the "really complete" notifications fit in naturally. But there are still a few complications resulting from the need to accurately tell an application _which_ buffers can be reused. 

An application doing zero-copy networking with io_uring will start by registering at least one completion context, using the `IORING_REGISTER_TX_CTX` registration operation. The context itself is a simple structure: 

```
struct io_uring_tx_ctx_register {
    	__u64 tag;
        };
```

The `tag` is a caller-chosen value used to identify this particular context in future zero-copy operations on the associated ring. There can be a maximum of 1024 contexts associated with the ring; user space should register them all with a single `IORING_REGISTER_TX_CTX` operation, passing the structures as an array. An attempt to register a second set of contexts will fail unless an intervening `IORING_UNREGISTER_TX_CTX` operation has been done to remove the first set. 

Zero-copy writes are initiated with the new `IORING_OP_SENDZC` operation. As usual, a set of buffers is passed to be written out to the socket (which must also be provided, obviously). Additionally, each zero-copy write must have a context associated with it, stored in the submission queue entry's `user_data` field. The context is specified as an index into the array of contexts that was registered previously (not as the tag associated with the context). These writes will use the kernel's zero-copy mechanism when possible and will "complete" in the usual way, with the usual result in the completion ring, perhaps while the supplied buffers are still in use. 

To know that the kernel is done with the buffers, the application must wait for the second notification informing it of that fact. Those notifications are not (by default) sent for every zero-copy operation that is submitted; instead, they are batched into "generations". Each completion context has a sequence number that starts at zero. Multiple operations can be associated with each generation; the notification for that generation is sent once all of the associated operations have truly completed. 

It is up to user space to tell the kernel when to move on to a new generation; that is done by setting the `IORING_SENDZC_FLUSH` flag in a zero-copy write request. The flag itself lives in the `ioprio` field of the submission queue entry. The presence of this flag indicates that the request being submitted is the last of the current generation; the next request will begin the new generation. Thus, if a separate done-with-the-buffers notification is needed for each write request, `IORING_SENDZC_FLUSH` should be set on every request. 

When a given generation completes, the notification will show up in the completion ring. The `user_data` field will contain the context tag, while the `res` field will hold the generation number. Once the notification arrives, the application will be able to safely reuse the buffers associated with that generation. 

The end result seems to be quite good; benchmarks included in the cover letter suggest that io_uring's zero-copy operations can perform more than 200% better than `MSG_ZEROCOPY`. Much of that improvement likely comes from the ability to use fixed buffers and files with io_uring, cutting out much of the per-operation overhead. Most applications won't see that kind of improvement, of course; they are not so heavily dominated by the cost of network transmission. If your business is providing the world with cat videos, though, zero-copy networking with io_uring is likely to be appealing. 

For now, the new zero-copy operations are meticulously undocumented. Begunkov has posted [a test application](<https://lwn.net/ml/linux-kernel/af0d6b3e-85d0-9b61-4acd-5eef1375f934@gmail.com/>) that can be read to see how the new interface is meant to be used. There have not been many comments on this version (the second) of this series. Perhaps that will change after the holidays, but it seems likely that this work is getting close to ready for inclusion.
