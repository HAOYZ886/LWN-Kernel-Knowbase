---
title: "Block-layer I/O polling"
url: https://lwn.net/Articles/663879/
date: "November 11, 2015"
category: Block layer
author: "By Jonathan Corbet November 11, 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 11, 2015

It has been said that the kernel's block I/O layer routinely steals ideas from the networking stack. In truth, good ideas move in both directions, but there can be no doubt that block I/O has become more like network I/O over the years. The details differ but, at the highest level, it's a matter of independent computers sending messages to each other over increasingly fast transports. So it is perhaps not surprising to see one of the network stack's oldest performance-improving techniques — I/O polling — show up in the block layer. 

In the networking world, I/O polling is called "NAPI" (for "new API"); LWN first [reported on it](<https://lwn.net/Articles/30107/>) in early 2003. NAPI allows the networking core to poll drivers for new packets, rather than having those drivers inject packets in response to interrupts from the interface hardware. Moving from an interrupt-driven mode to polling for performance reasons may seem counter-intuitive but, in a high-traffic situation, it makes sense. Servicing interrupts is expensive; it's also pointless if you know that there will be new packets available whenever you get around to looking for them. If the CPU has nothing else to do while waiting for packets, polling is also a good way to minimize latency. It will always be faster to watch for an arriving packet than to wait for the entire interrupt-handling machinery (in both hardware and software) to do its thing. 

I/O polling made no sense for the block layer as long as storage was dominated by rotating media. A computer can get a lot of work done by the time the disk head and platter move to the right position for the data of interest, and even the fastest drive can only generate hundreds of completion interrupts each second. Solid-state drives are different, though; I/O completion times are tiny and even a low-end drive can complete huge numbers of operations per second. With such a drive, the case for doing some other work while waiting for an I/O completion interrupt is rather weaker. 

How much weaker can be seen in the cover letter for the [polled block I/O patch set](<https://lwn.net/Articles/663543/>) from Jens Axboe and Christoph Hellwig. Using a sophisticated "read the device with `dd`" benchmark, Jens shows that, when polling is enabled, the throughput of an NVM Express device can nearly double. One might argue that this benchmark is designed to maximize the perceived performance benefit, but it also mirrors real-world usage patterns. A program doing synchronous reads from a block device, where it must wait for each read to complete before proceeding, is not an uncommon sight. 

(If one wants to quibble further with the results, more fertile ground may be found in this comment from Jens: ""Contrary to intuition, sometimes the slower devices benefit more, since the slower completion yields a deeper C-state on the processor."" The suggestion here is that polling is gaining some of its benefit by preventing the CPU from going into a sleep state; it would be interesting to see the results when power management is disabled.) 

The current patch set can only enable polling for devices driven via the multiqueue API. If a device is fast enough for polling to make sense, use of multiqueue I/O is probably indicated as well. Polling is controlled by the new `queue/io_poll` sysfs flag attached to each block device; the default is to not use polling. 

The first step in the patch series affects a fairly wide range of drivers, as it changes the prototype of the `make_request_fn()` to return a "cookie" identifying each submitted I/O operation: 

```
typedef unsigned int blk_qc_t;
        typedef blk_qc_t (make_request_fn) (struct request_queue *q,
        		      struct bio *bio);
```

The cookie returned by `make_request_fn()` can be anything, but the expected pattern is for drivers to use: 

```
blk_qc_t blk_tag_to_qc_t(unsigned int tag, unsigned int queue_num);
```

to construct the cookie from the queue number and the tag identifying the request. The special `BLK_QC_T_NONE` value can be used to indicate that no cookie exists. This change ripples through the block-driver subsystem, as each driver must be changed to reflect the new prototype regardless of whether it supports polling. Once that structure is in place, the special multiqueue `make_request_fn()` is changed to return the expected cookie. 

The core of the patch is the addition of a function to poll on the completion of a specific I/O request: 

```
bool blk_poll(struct request_queue *q, blk_qc_t cookie);
```

This function, in turn, calls a new driver-specific function added to the `blk_mq_ops` structure: 

```
typedef int (poll_fn)(struct blk_mq_hw_ctx *ctx, unsigned int tag);
```

This function should poll the status of the operation identified by `tag`, returning a nonzero value if that operation has completed. `blk_poll()` will call the driver-level `poll_fn()` repeatedly as long as the operation remains outstanding, no higher-priority process wants to run, and no signals are pending. A call to `blk_poll()` is added to the direct I/O implementation, so that synchronous, direct I/O will poll for completion whenever it is possible. Finally, the NVMe low-level driver gains a `poll_fn()` to actually implement the polling. 

The results are as described above: a large increase in I/O throughput. That is the case even though the NVMe implementation could stand some improvement: it currently leaves interrupts enabled, so I/O completion will interrupt the processor even when polling is in use. In any case, complete elimination of interrupts, as happens with NAPI, may be more difficult in the block context. A NAPI driver puts itself explicitly into the polling mode, and the actual polling is scheduled by the networking core. A block driver, instead, only knows that polling is in use when its `poll_fn()` is called, and that can be done by any process that is waiting for I/O. Since a block driver can never know that another `poll_fn()` call is coming, it must always be prepared to handle completion via interrupts. 

That said, this API is in an early state and may evolve considerably before it is considered production-ready. The main purpose for posting it now is to enable other developers to play with it — an objective that should be easy to achieve since this patch set was merged for the 4.4 release. As that playing takes place, the resulting experience should lead to improvements in the interface. And the process of streamlining the block layer to allow it to keep up with ever-faster storage devices will continue.
