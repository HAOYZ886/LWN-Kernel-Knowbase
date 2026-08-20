---
title: "Block layer: solid-state storage, timeouts, affinity, and more"
url: https://lwn.net/Articles/303270/
date: "October 15, 2008"
category: "Block layer-Block drivers; Device drivers-Block drivers"
author: "By Jonathan Corbet October 15, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 15, 2008

The 2.6.28 merge window has seen the addition of a number of changes to the block layer. Here's a summary of the new features and APIs which have gone in. 

### Solid-state storage devices

There are some enhancements aimed at improving the kernel's support of solid state storage devices. One of those, the discard API, has been [covered here](<https://lwn.net/Articles/293658>) before. This API allows high-level block subsystem users (filesystems) to indicate that a particular range of blocks no longer contains useful data. That allows the low-level device to incorporate those blocks into its garbage collection scheme and to stop worrying about their contents when performing wear leveling. 

Since the initial LWN article, though, the API has changed a little. The way to issue a discard request is now: 

```
int blkdev_issue_discard(struct block_device *bdev, sector_t sector,
    			     unsigned nr_sects);
```

The `end_io()` parameter seen in previous versions of the API is no longer present. There is no way for callers to know when the request completes, or, indeed, if the request completes at all. Since the caller is indicating a lack of interest in the given sectors, it really should not matter what the device does thereafter. 

There is a filesystem-level function for creating discard requests: 

```
static inline int sb_issue_discard(struct super_block *sb,
    				       sector_t block, 
    				       unsigned nr_blocks);
```

Here, the interface is expecting block numbers using the filesystem block size, rather than 512-byte sectors. 

User-space programs can issue discard requests with the new `BLKDISCARD` `ioctl()` call. Needless to say, such operations should be done with care; about the only logical user of this `ioctl()` would be `mkfs` programs. 

Block drivers which support discard requests will provide a suitable function to the block layer: 

```
typedef int (prepare_discard_fn) (struct request_queue *queue, 
        	    			      struct request *rq);
    
        void blk_queue_set_discard(struct request_queue *q, 
        	                       prepare_discard_fn *dfn);
```

In the absence of a "prepare discard" function, discard requests for the device will fail. 

The block layer has also added a flag by which drivers can indicate that a device is not rotating storage, and, thus, does not suffer from seek delays. By setting `QUEUE_FLAG_NONROT` (with `queue_flag_set()` or `queue_flag_set_unlocked()`), a driver tells the block layer that it is working with a solid state device. I/O schedulers can use that information to avoid plugging the queue - a useful technique for combining requests to rotating storage devices, but a useless operation when there is no seek penalty to avoid. 

### Request affinity

On large, multiprocessor systems, there can be a performance benefit to ensuring that all processing of a block I/O request happens on the same CPU. In particular, data associated with a given request is most likely to be found in the cache of the CPU which originated that request, so it makes sense to perform the request postprocessing on that same CPU. With 2.6.28, sysfs entries for block devices will include an `rq_affinity` variable. If it is set to a non-zero value, CPU affinity will be turned on for that device. According to the patch changelog, turning this feature on can reduce system time by 20-40% on some benchmarks. 

### Timeout handling

Robust device drivers typically have to be written to handle cases where devices fail to complete operations they have been instructed to do. In a few cases, higher-level code helps with this task; the networking layer, for example, can track outgoing packets and let a driver know when a transmit operation has taken too long. In most other drivers, though, it's up to the driver itself to notice when an operation seems to be taking too long. 

Like the network subsystem, the block layer manages queues of requested operations. As of 2.6.28 the block layer will, again like networking, have a mechanism for notifying drivers about request timeouts; that, in turn, will allow a bunch of timeout-related code to be removed from the lower layers. Timeout handling in the block layer can be more complex, though, and the associated API reflects that complexity. 

A block driver must register a function to handle timed-out requests: 

```
typedef enum blk_eh_timer_return (rq_timed_out_fn)(struct request *);
    
        void blk_queue_rq_timed_out(struct request_queue *q, 
    				rq_timed_out_fn *fn);
```

The amount of time a request should be outstanding before timing out is set up with: 

```
void blk_queue_rq_timeout(struct request_queue *q, 
        	 		      unsigned int timeout);
```

The tracking of per-request timeouts is done within the block layer; the timer for any individual request is started when that request is dispatched to the driver by the I/O scheduler. Should a request fail to complete before the timeout period passes, the driver's timeout function will be called with a pointer to the languishing request. The driver then can do one of three things: 

  * Figure out that, in fact, the request was completed as expected, but that completion had not been noticed by the driver. A dropped interrupt could bring out such a situation, for example. In this case, the driver returns `BLK_EH_HANDLED`, and the request will be marked as completed. 

  * Decide that the request needs more time, perhaps because it has been re-issued by the driver. A `BLK_EH_RESET_TIMER` will start the timer again for this request. 

  * Punt and return `BLK_EH_NOT_HANDLED`. The block layer currently does nothing at all when it gets this return code; future plans appear to include aborting the request within the block layer when this return value is encountered. 

If things look bad, the driver may decide to abort any outstanding requests, reset the device, and start over. There are a couple of new functions which can help with this task: 

```
void blk_abort_request(struct request *req);
        void blk_abort_queue(struct request_queue *q);
```

These functions will abort the given request, or all requests on the queue, as appropriate. Part of that process involves calling the driver's timeout handler for each aborted request. 

### Other changes in brief

Some other block-layer changes include: 

  * The handling of minor numbers has been changed, allowing disks to have an essentially unbounded number of partitions. The cost of this change is that minor numbers may be attached to a different major number, and they might not all be contiguous; for this reason, drivers must set the `GENHD_FL_EXT_DEVT` flag before the extended numbers will be used. See [this article](<https://lwn.net/Articles/290141/>) for more information on this change. 

  * The prototypes of `blk_rq_map_user()` and `blk_rq_map_user_iov()` have changed; there is now a `gfp_mask` parameter. This allows these functions to be used in atomic context. 

  * `kblockd_schedule_work()` has an additional parameter specifying the relevant request queue. 

  * The new function `bio_kmalloc()` behaves much like `bio_alloc()`, but it does not use a mempool to guarantee allocations and can thus fail. 

It is, all told, one of the busier development cycles for the block layer in recent times.
