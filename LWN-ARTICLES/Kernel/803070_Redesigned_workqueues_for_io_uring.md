---
title: Redesigned workqueues for io_uring
url: https://lwn.net/Articles/803070/
date: "October 25, 2019"
category: Workqueues; io uring
author: "By Jonathan Corbet October 25, 2019"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 25, 2019

The [io_uring mechanism](<https://lwn.net/Articles/776703/>) is a relatively new interface for asynchronous I/O; it first appeared in the 5.1 kernel in May. Since then, though, it has quickly grown in capabilities and in users; now it appears that it is outgrowing some of the kernel infrastructure that supports it. Thus, we have a proposal from Jens Axboe (the io_uring maintainer) for [a new workqueue subsystem](<https://lwn.net/ml/linux-block/20191024134439.28498-1-axboe@kernel.dk/>) for io_uring that hints at some interesting plans for the future. 

[Workqueues](<https://www.kernel.org/doc/html/latest/core-api/workqueue.html>) are used by many kernel subsystems to run work asynchronously in process context. Over the years, workqueues have been extensively tweaked to provide the features needed by the kernel and to keep queued work requests from running concurrently on the same processor and contending with each other for CPU time. They have been relatively stable for a while, indicating that they do what is needed most of the time. 

The io_uring mechanism is all about allowing user space to create asynchronous threads of execution, so it's not surprising that workqueues are extensively used there. Over time, though, some of the limitations of workqueues have become apparent in this context. Workqueues are, to a great extent, about ceding control over where and when the work functions are executed, but io_uring would benefit from a higher degree of control over how that work is done. Thus, the new mechanism, called "io-wq". 

One of the new workqueues (an "`io_wq`") is created with: 

```
struct io_wq *io_wq_create(unsigned concurrency, struct mm_struct *mm);
```

where `concurrency` is the maximum number of worker threads that can be running on any given NUMA node, and `mm` is the memory-management context associated with this queue. In io_uring, one of these workqueues will be created for each `io_uring_setup()` call, and `mm` will point to the calling process's `mm_struct` structure. Associating the `mm_struct` with the `io_wq` in this way makes a number of the memory-management issues easier; the existing workqueue mechanism does not maintain this association, even when private workqueues are created. 

The `io_wq` may be destroyed by passing it to `io_wq_destroy()`. 

To defer work to an `io_wq`, one starts by filling out one of these structures: 

```
struct io_wq_work {
    	struct list_head list;
    	void (*func)(struct io_wq_work **);
    	unsigned flags;
        };
```

The main thing to do is to set `func` to the function that should be called to actually do the work; `flags` should be set to zero. The item can then be queued with either of: 

```
void io_wq_enqueue(struct io_wq *wq, struct io_wq_work *work);
        void io_wq_enqueue_hashed(struct io_wq *wq, struct io_wq_work *work, 
    			      void *val);
```

A call to `io_wq_enqueue()` adds the `work` to the queue for future execution. The `io_wq_enqueue_hashed()` variant, instead, is one of the reasons for the creation of new mechanism; it guarantees that no two jobs enqueued with the same `val` will run concurrently. If an application submits multiple buffered I/O requests for a single file, they should not be run concurrently or they are likely to just block each other via lock contention. Buffered I/O on different files can and should run concurrently, though. "Hashed" work entries make it easy for io_uring to manage that concurrency in an optimal way. 

Passing an `io_wq` to `io_wq_flush()` will cause the calling thread to block until all pending work items have left the queue. Note that this does _not_ mean that those items have completed, only that they have started. 

Cancellation is another motivation for io-wq. The io_uring mechanism has to allow user space to cancel pending requests, meaning that it must be possible to cancel io-wq work requests in a predictable way. In current kernels, cancellation of requests on network sockets can occasionally lead to deadlocks; users tend to find this kind of behavior less amusing than one might think, so a better solution is needed. The new cancellation functions are: 

```
void io_wq_cancel_all(struct io_wq *wq);
        enum io_wq_cancel io_wq_cancel_work(struct io_wq *wq,
        					struct io_wq_work *cwork);
```

A call to the first function will cancel all outstanding operations on the given `io_wq`; the second one will cancel only the specified work request. Either way, this is done by sending a `SIGINT` signal to each running worker thread; the function will return after the signals have been sent without waiting for the worker threads to respond. For `io_wq_cancel_work()`, the return value will be `IO_WQ_CANCEL_OK` (the request was canceled before starting), `IO_WQ_CANCEL_RUNNING` (the request was running and the signal was sent), or `IO_WQ_CANCEL_NOTFOUND` (the request wasn't found, meaning it had already completed). 

That is about it for the io-wq API. It is not clear that there would be benefits for any other kernel subsystem to move to this mechanism, so io_uring may remain the only user for some time. An improved io_uring will be enough for many users to celebrate, though. 

That said, there may be more coming. Long-time LWN readers may remember a series of discussions in 2007 for an in-kernel mechanism called, at times, [fibrils](<https://lwn.net/Articles/219954/>), [threadlets](<https://lwn.net/Articles/223899/>), or [syslets](<https://lwn.net/Articles/236206/>). Regardless of the name, this mechanism was intended to improve asynchronous I/O support, but there was another motive as well: to allow user space to run _any_ system call asynchronously. None of those mechanisms reached a point of being seriously considered for merging, but it seems that they were not forgotten. In [patch two](<https://lwn.net/ml/linux-block/20191024134439.28498-3-axboe@kernel.dk/>) of the series, Axboe notes that using io-wq in io_uring ""gets us one step closer to adding async support for any system call"". It thus seems that we can expect io_uring to develop the capabilities that were envisioned almost 13 years ago. Stay tuned for further developments.
