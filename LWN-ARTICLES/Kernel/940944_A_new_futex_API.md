---
title: A new futex API
url: https://lwn.net/Articles/940944/
date: "August 14, 2023"
category: Futex
author: "By Jonathan Corbet August 14, 2023"
---

By **Jonathan Corbet**  
August 14, 2023

The Linux fast user-space mutex ("futex") subsystem debuted with the 2.6.0 kernel; it provides a mechanism that can be used to implement user-space locking. Since futexes avoid calling into the kernel whenever possible, they can indeed be fast, especially in the uncontended case. The API used to access futexes has never been seen as one of Linux's strongest points, though, so there has long been a desire to improve it. [This patch series](<https://lwn.net/ml/linux-kernel/20230807121843.710612856@infradead.org/>) from Peter Zijlstra shows what the future of futexes may look like. 

A futex is a 32-bit value stored in user-space memory that is, presumably, shared between at least two threads or processes. When used as a lock, a futex can be acquired with a single compare-and-swap instruction, without kernel involvement. The kernel comes into the picture, though, in the contended case, where a thread must block until a futex becomes available. Waiting for a futex and waking threads that are waiting are some of the features provided by the [`futex()`](<https://man7.org/linux/man-pages/man2/futex.2.html>) system call. 

`futex()` is a multiplexed system call, meaning that it performs a number of unrelated operations depending on its arguments. Over the years, as futex functionality has grown, this system call has become complex and unwieldy, to put it mildly. It is difficult to use, and difficult to extend further. This API has, among other things, slowed the addition of features that developers would like to see. 

For some years, there has been talk of splitting the futex API into a set of more focused system calls; LWN [covered one such discussion](<https://lwn.net/Articles/823513/>) in 2020. Thus far, though, actual progress in this direction has been limited to [the addition of `futex_waitv()`](<https://lwn.net/Articles/866112/>) to the 5.16 release in early 2022; work on futexes seemingly stalled after that. With the current patch set, Zijlstra appears to be trying to restart this project and bring a better futex API to the mainline. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

This patch series includes three new system calls: 

```
int futex_wait(void *addr, unsigned long val, unsigned long mask, unsigned int flags,
        		   struct __kernel_timespec *timeout, clockid_t clockid);
    
        int futex_wake(void *addr, unsigned long mask, int nr, unsigned int flags);
    
        int futex_requeue(struct futex_waitv *waiters, unsigned int flags, int nr_wake,
    		      int nr_requeue);
```

For the most part, so far, these system calls are just new interfaces to existing functionality. `futex_wait()` is the same as `futex(FUTEX_WAIT_BITSET)`; it will cause the calling thread to wait on the futex stored at `addr`, assuming that futex contains `val` at the time of the call. The `mask` will be stored in the kernel's context for this thread. `futex_wake()` mirrors the `FUTEX_WAKE_BITSET` operation, using `mask` to identify which waiter(s) to wake. `futex_requeue()` is another interface to `FUTEX_CMP_REQUEUE`, which can wake some waiters and requeue others onto a different futex. 

There is also some new functionality that is made available via the new API, generally using the `flags` argument. One of the new flags is `FUTEX2_NUMA`, which is intended to improve performance on NUMA systems. User space is in control of the placement of its futexes and can, thus, ensure that they live on a NUMA node that is close to the threads that are using it. But the kernel maintains its own data structures for futexes that are being waited on; poor placement of those structures can slow everything down. This problem was [mentioned here](<https://lwn.net/Articles/685769/>) in 2016, but still lacks a solution in mainline kernels. 

As noted above, a futex is a 32-bit value. When the `FUTEX2_NUMA` flag is provided, though, the futex(es) referred to are, instead, interpreted as: 

```
struct futex_numa_32 {
    	u32 val;
    	u32 node;
        };
```

(Though this structure does not actually appear in this form in the kernel source). 

The `val` field is the same old 32-bit quantity, while `node` is the number of the NUMA node on which the kernel should allocate its own data structures. The `node` value can also be set to all-ones (`~0`), in which case the current NUMA node will be used, and the `node` value will be updated accordingly by the kernel. The [patch](<https://lwn.net/ml/linux-kernel/20230807123323.504975124@infradead.org/>) adding this structure admonishes: ""If userspace corrupts the node value between WAIT and WAKE, the futex will not be found and no wakeup will happen"". 

The `FUTEX2_NUMA` flag thus gives user space some control over the placement of associated memory within the kernel. If this flag does not appear, these allocations will be spread across all of the nodes of the system (as is done with futexes in current kernels). 

One other change in this series is reflected by another set of new flags. There has long been a desire to get away from the 32-bit requirement for futexes. Often, only a single bit is used; a smaller size would reduce waste and (more importantly) allow more futexes to be crammed into a single cache line. To accommodate this need, the new system calls require a `flags` value indicating the size of the futex(es) to be operated on; it can be one of `FUTEX2_SIZE_U8`, `FUTEX2_SIZE_U16`, or `FUTEX2_SIZE_U32`. There is also a `FUTEX2_SIZE_U64` flag defined, but 64-bit futexes are not implemented in the current patch set. 

When `FUTEX2_NUMA` is used, the node number has the same size as the futex value itself. Thus, a futex operation specifying `FUTEX2_NUMA|FUTEX2_SIZE_U8` will provide an eight-bit node number, which could be modeled as: 

```
struct futex_numa_8 {
        	u8 val;
    	u8 node;
        };
```

The end result of this work is a set of incremental improvements that cleans up the futex API and provides some functionality that developers have been asking for. Review comments have mostly been focused on relatively minor details, suggesting that there may not be much in the way of getting these changes merged into the mainline. Of course, the job does not end with these patches; there is still a lot of functionality provided by `futex()`, including priority inheritance, that is not available with the new API. But this work should make the common cases easier for developers to work with and might even, someday, lead to support for futexes in the GNU C library.
