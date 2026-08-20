---
title: Explicit block device plugging
url: https://lwn.net/Articles/438256/
date: "April 13, 2011"
category: "Block layer-Plugging"
author: "April 13, 2011 This article was contributed by Jens Axboe"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

April 13, 2011

This article was contributed by Jens Axboe

Since the dawn of time, or for at least as long as I have been involved, the Linux kernel has deployed a concept called "plugging" on block devices. When I/O is queued to an empty device, that device enters a plugged state. This means that I/O isn't immediately dispatched to the low level device driver, instead it is held back by this plug. When a process is going to wait on the I/O to finish, the device is unplugged and request dispatching to the device driver is started. The idea behind plugging is to allow a buildup of requests to better utilize the hardware and to allow merging of sequential requests into one single larger request. The latter is an especially big win on most hardware; writing or reading bigger chunks of data at the time usually yields good improvements in bandwidth. With the release of the 2.6.39-rc1 kernel, block device plugging was drastically changed. Before we go into that, lets take a historic look at how plugging has evolved. 

Back in the early days, plugging a device involved global state. This was before SMP scalability was an issue, and having global state made it easier to handle the unplugging. If a process was about to block for I/O, any plugged device was simply unplugged. This scheme persisted in pretty much the same form until the early versions of the 2.6 kernel, where it began to severely impact SMP scalability on I/O-heavy workloads. 

In response to this problem, the plug state was [turned into a per-device entity](<https://lwn.net/Articles/75233/>) in 2004\. This scaled well, but now you suddenly had no way to unplug all devices when going to sleep waiting for page I/O. This meant that the virtual memory subsystem had to be able to unplug the specific device that would be servicing page I/O. A special hack was added for this: `sync_page()` in `struct address_space_operations`; this hook would unplug the device of interest. 

If you have a more complicated I/O setup with device mapper or RAID components, those layers would in turn unplug any lower-level device. The unplug event would thus percolate down the stack. Some heuristics were also added to auto-unplug the device if a certain depth of requests had been added, or if some period of time had passed before the unplug event was seen. With the asymmetric nature of plugging where the device was automatically plugged but had to be explicitly unplugged, we've had our fair share of I/O stall bugs in the kernel. While crude, the auto-unplug would at least ensure that we would chuck along if someone missed an unplug call after I/O submission. 

With really fast devices hitting the market, once again plugging had become a scalability problem and hacks were again added to avoid this. Essentially we disabled plugging on solid-state devices that were able to do queueing. While plugging originally was a good win, it was time to reevaluate things. The asymmetric nature of the API was always ugly and a source of bugs, and the `sync_page()` hook was always hated by the memory management people. The time had come to rewrite the whole thing. 

The primary use of plugging was to allow an I/O submitter to send down multiple pieces of I/O before handing it to the device. Instead of maintaining these I/O fragments as shared state in the device, a new on-stack structure was created to contain this I/O for a short period, allowing the submitter to build up a small queue of related requests. The state is now tracked in `struct blk_plug`, which is little more than a linked list and a `should_sort` flag informing `blk_finish_plug()` whether or not to sort this list before flushing the I/O. We'll come back to that later. 

```
struct blk_plug {
    		unsigned long magic;
    		struct list_head list;
    		unsigned int should_sort;
    	};
```

The magic member is a temporary addition to detect uninitialized use cases, it will eventually be removed. The new API to do this is straightforward and simple to use: 

```
struct blk_plug plug;
    
    	blk_start_plug(&plug);
    	submit_batch_of_io();
    	blk_finish_plug(&plug);
```

`blk_start_plug()` takes care of initializing the structure and tracking it inside the task structure of the current process. The latter is important to be able to automatically flush the queued I/O should the task end up blocking between the call to `blk_start_plug()` and `blk_finish_plug()`. If that happens, we want to ensure that pending I/O is sent off to the devices immediately. This is important from a performance perspective, but also to ensure that we don't deadlock. If the task is blocking for a memory allocation, memory management reclaim could end up wanting to free a page belonging to a request that is currently residing on our private plug. Similarly, the caller may itself end up waiting for some of the plugged I/O to finish. By flushing this list when the process goes to sleep, we avoid these types of deadlocks. 

If `blk_start_plug()` is called and the task already has a plug structure registered, it is simply ignored. This can happen in cases where the upper layers plug for submitting a series of I/O, and further down in the call chain someone else does the same. I/O submitted without the knowledge of the original plugger will thus end up on the originally assigned plug, and be flushed whenever the original caller ends the plug by calling `blk_finish_plug()`, or if some part of the call path goes to sleep or is scheduled out. 

Since the plug state is now device agnostic, we may end up in a situation where multiple devices have pending I/O on this plug list. These may end up on the plug list in an interleaved fashion, potentially causing `blk_finish_plug()` to grab and release the related queue locks multiple times. To avoid this problem, a `should_sort` flag in the `blk_plug` structure is used to keep track of whether we have I/O belonging to more than I/O distinct queue pending. If we do, the list is sorted to group identical queues together. This scales better than grabbing and releasing the same locks multiple times. 

With this new scheme in place, the device need no longer be notified of unplug events. The queue `unplug_fn()` used to exist for this purpose alone, it has now been removed. For most drivers it is safe to just remove this hook and the related code. However, some drivers used plugging to delay I/O operations in response to resource shortages. One example of that was the SCSI midlayer; if we failed to map a new SCSI request due to a memory shortage, the queue was plugged to ensure that we would call back into the dispatch functions later on. Since this mechanism no longer exists, a similar API has been provided for such use cases. Drivers may now use blk_delay_queue() for this: 

```
blk_delay_queue(queue, delay_in_msecs);
```

The block layer will re-invoke request queueing after the specified number of milliseconds have passed. It will be invoked from process context, just as it would have been with the unplug event. `blk_delay_queue()` honors the queue stopped state, so if `blk_stop_queue()` was called before `blk_delay_queue()`, or if is called after the fact but before the delay has passed, the request handler will not be invoked. `blk_delay_queue()` must only be used for conditions where the caller doesn't necessarily know when that condition will change states. If resources internal to the driver cause it to need to halt operations for a while, it is more efficient to use `blk_stop_queue()` and `blk_start_queue()` to manage those directly. 

These changes have been merged for the 2.6.39 kernel. While a few problems have been found (and fixed), it would appear that the plugging changes have been integrated without greatly disturbing Linus's calm development cycle.
