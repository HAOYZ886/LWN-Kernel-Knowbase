---
title: "Avoiding blocking file-name lookups"
url: https://lwn.net/Articles/843163/
date: "January 21, 2021"
category: "Filesystems-Virtual filesystem layer; System calls-openat2; io uring"
author: "By Jonathan Corbet January 21, 2021"
---

By **Jonathan Corbet**  
January 21, 2021

As a general rule, when one attempts to open a file with a system call like [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>), the expectation is that the call will not return until the job is done. But there are times where the desire to open the file is conditional on being able to open it immediately, without blocking. Linux has never supported that mode well, but that may be about to change with [this patch set](<https://lwn.net/ml/linux-fsdevel/20201217161911.743222-1-axboe@kernel.dk/>) from Jens Axboe. 

Opening a file can be a complex operation. Simply resolving the name of the file can require traversing a series of directories that may be located on different filesystems; each step may also require performing I/O and taking locks to serialize changes that might otherwise create unwelcome surprises. Once the file has been found, there may be more I/O required to perform the open itself. Each of these steps has the potential to block the opening task for an unknown period of time. 

Axboe's patch set creates a new internal flag called `LOOKUP_CACHED`, which is then made available to callers of `openat2()` as `RESOLVE_CACHED`. This flag requests the kernel to only carry the open to completion if that can be done using only data that is cached in memory — without performing I/O, in other words. If it becomes clear during the attempt that I/O would be required, the `openat2()` call will fail with an `EAGAIN` error. The caller can then retry the operation without `RESOLVE_CACHED` — in a setting where blocking is tolerable — to successfully open the file. 

One might well wonder what this new option is for; it is not often that a program needs to open a file only if it can be done quickly. The motivating use case here is in the [io_uring subsystem](<https://lwn.net/Articles/776703/>), which has grown considerably in the two years since it first appeared. The core purpose for io_uring is performing asynchronous I/O, but it increasingly has the ability to run other system calls — including `openat2()` — intermixed with I/O operations. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Many of those other system calls were never designed with asynchronous use in mind, so they will happily block if need be; that is something that io_uring cannot allow, since it would block the handling of other operations as well. So io_uring creates a separate kernel thread to run system calls that might block at inopportune times. That effectively makes those calls asynchronous, but at a cost: moving ring operations into a separate thread can slow execution considerably. For an operation that _can_ be carried out using only cached data, the overhead of shifting to another thread becomes a dominant performance factor. 

The solution is to use this new `LOOKUP_CACHED` flag. Whenever an open operation is called for in io_uring, an attempt will be made to execute it directly with `LOOKUP_CACHED`. If that works, all is well and the operation completes successfully; otherwise, it will be pushed off to a thread and retried without `LOOKUP_CACHED` as before. According to Axboe, an open-heavy benchmark will run nearly three times faster if all of the necessary data is already cached. 

Another question that might come to mind is: why was the existing `O_NONBLOCK` flag not used for this purpose? There may be a number of reasons, but one that jumps out is that `O_NONBLOCK` applies to the resulting file descriptor for its entire life; _all_ operations performed on that descriptor will (potentially, at least) be non-blocking. The `RESOLVE_CACHED` flag, instead, applies only to the opening of the file. 

Making `open()` calls be truly non-blocking has been a challenge for kernel developers for longer than Linux has existed. Given that, it can be surprising to see how small this patch set is; there was little that needed to be done. This work has benefited greatly from the [RCU walk](<https://lwn.net/Articles/419811/>) mechanism that was added ten years ago. The purpose then was to make file-name lookup operations faster by avoiding taking locks whenever possible; that required creating a lookup path that would bail out anytime an operation might block. Normally, a lookup operation will be retried with the slow path if a RCU-walk lookup fails; the `LOOKUP_CACHED` patch just has to restrict lookups to the RCU-walk path to get the needed result. 

This patch set is in its fourth revision. Its fate can perhaps be foretold from [this comment](<https://lwn.net/ml/linux-fsdevel/CAHk-=wjxQOBVZiX-OD9YC1ZkA-N4tG7sjtkWApY8Rtz4gb_k6Q@mail.gmail.com/>) by Linus Torvalds: 

Despite having seen the patch series so many times now, I'm still just really impressed by how small and neat it is, considering the above numbers, and considering just how problematic this case was historically (ie I remember all the discussions we had about nonblocking opens back in the days). 

So I continue to go "this is the RightWay(tm)" just from that. 

In the absence of surprising problems, it seems likely that little will block this work from landing in the mainline as soon as the 5.12 merge window. Whether application developers will find a use for `RESOLVE_CACHED` remains to be seen, but io_uring users should benefit from this feature from the outset.
