---
title: Block layer request queue API changes
url: https://lwn.net/Articles/333620/
date: "May 18, 2009"
category: "Block layer-Block drivers"
author: "By Jonathan Corbet May 18, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 18, 2009

Prior to the 2.6 kernel series, the Linux block layer was somewhat simplistic and inflexible; it showed a lot of history from the early days of the Linux kernel. With the 2.5 development series came a complete rewrite; there have, of course, been a great many changes since then as well. But there are still bits of history to be found in the Linux block API. If Tejun Heo has his way, some of that history will be gone in the near future. 

The standard way for a block driver to gain access to the next I/O request in the queue is with a call to: 

```
struct request *elv_next_request(struct request_queue *queue);
```

This function returns the request which is, in the I/O scheduler's opinion, the best one to execute next. An interesting feature of `elv_next_request()` is that it leaves the request on the queue; two calls to `elv_next_request()` in quick succession will return pointers to the same request. A block driver can explicitly remove the request from the queue with a call to `blkdev_dequeue_request()`, but that step is not necessary. If a request remains at the head of the queue when the block driver indicates that it has been completed, the block layer will dequeue the request at that time. 

Leaving the request on the queue is a throwback to the very early days, when requests were handled one at a time - often a single sector at a time. By hiding the queuing details, the block layer made life easier for simple block drivers. But this apparent simplicity comes at a cost: it complicates the block API and makes it impossible for the block layer to know when processing of a request has begun. So it's not possible to do reliable request timing when drivers work on requests which remain on the queue. 

This feature is also increasingly useless. Any contemporary driver worth its salt will process multiple requests at once; that, in return, requires that the driver dequeue requests and keep track of them itself. So few drivers that people actually care about use the process-on-queue model. Given that, Tejun has come to the conclusion that processing on-queue requests is an idea whose time has passed. He has posted [a lengthy patch series](<http://lwn.net/Articles/332512/>) to make it go away. 

The bulk of the patches are concerned with converting all drivers over to the "dequeue the request first" mode of operation. Typically that's just a matter of adding a `blkdev_dequeue_request()` call in the right place. A few places (the IDE subsystem, for example) are a bit more complex, but, for the most part, the changes are straightforward. 

Once that has been done, the patch series culminates with a set of API changes. There is no more `elv_next_request()`; instead, a driver wanting to look at a request without dequeueing it will call: 

```
struct request *blk_peek_request(struct request_queue *queue);
```

Following that, a request can be dequeued with a call to `blk_start_request()`, which replaces `blkdev_dequeue_request()`: 

```
void blk_start_request(struct request *req);
```

In addition to removing the request from the queue, `blk_start_request()` will start a timer for the request, allowing it to eventually respond if completion is never signaled. Most of the time, though, drivers will just call: 

```
struct request *blk_fetch_request(struct request_queue *q);
```

which is a combination of `blk_peek_request()` and `blk_start_request()`. 

There is one other, under-the-hood change which goes along with the above: any attempt to complete a request which remains on the request queue will oops the system. One can think of this as a very clear message that on-queue processing is no longer considered to be the right thing to do in the Linux kernel. That, in turn, is part of the motivation for the API changes, which, for the most part, are just name changes: Tejun wants to be sure that maintainers of out-of-tree block drivers will notice that something has changed and respond accordingly. 

These patches have been through a couple of rounds of review. Nothing is ever certain, but it's entirely possible that this set of changes could go in for the 2.6.31 kernel.
