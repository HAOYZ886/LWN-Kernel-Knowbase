---
title: Asynchronous block loop I/O
url: https://lwn.net/Articles/535034/
date: "January 30, 2013"
category: "Block layer-Loopback device; Loopback device"
author: "By Jonathan Corbet January 30, 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 30, 2013

The kernel's block loop driver has a conceptually simple job: take a file located in a filesystem somewhere and present it as a block device that can contain a filesystem of its own. It can be used to manipulate filesystem images; it is also useful for the management of filesystems for virtualized guests. Despite having had some optimization effort applied to it, the loop driver in current kernels is not as fast as some would like it to be. But that situation may be about to change, thanks to an old patch set that has been revived and prepared for merging in a near-future development cycle. 

As a block driver, the loop driver accepts I/O requests described by `struct bio` (or "BIO") structures; it then maps each request to a suitable block offset in the file serving as backing store and issues I/O requests to perform the desired operations on that file. Each loop device has its own thread, which, at its core, runs a loop like this: 

```
while (1) {
    	wait_for_work();
    	bio = dequeue_a_request()
    	execute_request(bio);
        }
```

(The actual code can be seen in `drivers/block/loop.c`.) This code certainly works, but it has an important shortcoming: it performs I/O in a synchronous, single-threaded manner. Block I/O is normally done asynchronously when possible; write operations, in particular, can be done in parallel with other work. In the loop above, though, a single, slow read operation can hold up many other requests, and there is no ability for the block layer or the I/O device itself to optimize the ordering of requests. As a result, the performance of loop I/O traffic is not what it could be. 

In 2009, Zach Brown set out to fix this problem by changing the loop driver to execute multiple, asynchronous requests at the same time. That work fell by the wayside when other priorities took over Zach's time, so his patches were never merged. More recently, Dave Kleikamp has taken over this patch set, ported it to current kernels, and added support to more filesystems. As a result, this patch set may be getting close to being ready to go into the mainline. 

At the highest level, the goal of this patch set is to use the kernel's existing asynchronous I/O (AIO) mechanism in the loop driver. Getting there takes a surprising amount of work, though; the AIO subsystem was written to manage user-space requests and is not an easy fit for kernel-generated operations. To make these subsystems work together, the [30-part patch set](<https://lwn.net/Articles/534928/>) takes a bottom-up approach to the problem. 

The AIO code is based around a couple of structures, one of which is `struct iovec`: 

```
struct iovec {
    	void __user *iov_base;
    	__kernel_size_t iov_len;
        };
```

This structure is used by user-space programs to describe a segment of an I/O operation; it is part of the user-space API and cannot be changed. Associated with this structure is the internal `iov_iter` structure: 

```
struct iov_iter {
    	const struct iovec *iov;
    	unsigned long nr_segs;
    	size_t iov_offset;
    	size_t count;
        };
```

This structure (defined in `<linux/fs.h>`) is used by the kernel to track progress working through an array of `iovec` structures. 

Any kernel code needing to submit asynchronous I/O needs to express it in terms of these structures. The problem, from the perspective of the loop driver, is that `struct iovec` deals with user-space addresses. But the BIO structures representing block I/O operations deal with physical addresses in the form of `struct page` pointers. So there is an impedance mismatch between the two subsystems that makes AIO unusable for the loop driver. 

Fixing that involves changing the way `struct iov_iter` works. The `iov` pointer becomes a generic pointer called `data` that can point to an array of `iovec` structures (as before) or, instead, an array of kernel-supplied BIO structures. Direct access to structure members by kernel code is discouraged in favor of a set of defined accessor operations; the `iov_iter` structure itself gains a pointer to an operations structure that can be changed depending on whether `iovec` or `bio` structures are in use. The end result is an enhanced `iov_iter` structure and surrounding support code that allows AIO operations to be expressed in either user-space (`struct iovec`) or kernel-space (`struct bio`) terms. Quite a bit of code using this structure must be adapted to use the new accessor functions; at the higher levels, code that worked directly with `iovec` structures is changed to work with the `iov_iter` interface instead. 

The next step is to make it possible to pass `iov_iter` structures directly into filesystem code. That is done by adding two more functions to the (already large) `file_operations` structure: 

```
ssize_t (*read_iter) (struct kiocb *, struct iov_iter *, loff_t);
        ssize_t (*write_iter) (struct kiocb *, struct iov_iter *, loff_t);
```

These functions are meant to work much like the existing `aio_read()` and `aio_write()` functions, except that they work with `iov_iter` structures rather than with `iovec` structures directly. A filesystem supporting the new operations must be able to cope with I/O requests expressed directly in BIO structures — usually just a matter of bypassing the page-locking and mapping operations required for user-space addresses. If these new operations are provided, the `aio_*()` functions will never be called and can be removed. 

After that, the patch set adds a new interface to make it easy for kernel code to submit asynchronous I/O operations. In short, it's a matter of allocating an I/O control block with: 

```
struct kiocb *aio_kernel_alloc(gfp_t gfp);
```

That block is filled in with the relevant information describing the desired operation and a pointer to a completion callback, then handed off to the AIO subsystem with: 

```
int aio_kernel_submit(struct kiocb *iocb);
```

Once the operation is complete, the completion function is called to inform the submitter of the final status. 

A substantial portion of the patch set is dedicated to converting filesystems to provide `read_iter()` and `write_iter()` functions. In most cases the patches are relatively small; most of the real work is done in generic code, so it is mostly a matter of changing declared types and making use of some of the new `iov_iter` accessor functions. See [the ext4 patch](<https://lwn.net/Articles/535178/>) for an example of what needs to be done. 

With all that infrastructural work done, actually speeding up the loop driver becomes straightforward. If the backing store for a given loop device implements the new operations, the loop driver will use `aio_kernel_submit()` for each incoming I/O request. As a result, requests can be run in parallel with, one hopes, a significant improvement in performance. 

The patch set has been through several rounds of review, and most of the concerns raised would appear to have been addressed. Dave is now asking that it be included in the linux-next tree, suggesting that he intends to push it into the mainline during the 3.9 or 3.10 development cycle. Quite a bit of kernel code will be changed in the process, but almost no differences should be visible from user space — except that block loop devices will run a lot faster than they used to.
