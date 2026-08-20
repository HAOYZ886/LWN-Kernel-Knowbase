---
title: Some block layer patches
url: https://lwn.net/Articles/157208/
date: "October 26, 2005"
category: Block layer; Elevator; Multipath IO; Write barriers
author: 
---

Lest LWN readers think that all of the development activity is currently centered around memory management issues, it is worth pointing out that some significant patches to the block subsystem are circulating as well. Here is a quick summary. 

Linux I/O schedulers are charged with presenting I/O requests to block devices in an optimal order. There are currently four schedulers in the kernel, each with a different notion of "optimal." All of them, however, maintain a "dispatch queue," being the list of requests which have been selected for submission to the device. Each scheduler currently maintains its own dispatch queue. 

Tejun Heo has decided that the proliferation of dispatch queues is a wasteful duplication of code, so he has implemented a [generic dispatch queue](<http://lwn.net/Articles/156273/>) to bring things back together. The unification of the dispatch queues helps to ensure that all I/O schedulers implement queues with the same semantics. It also simplifies the schedulers by freeing them of the need to deal with non-filesystem requests. In general, the developers have been heard to say, recently, that the block subsystem is not really about block devices; it is, instead, a generic message queueing mechanism. The generic dispatch queue code helps to take things in that direction. 

Tejun Heo has also [reimplemented the I/O barrier code](<http://lwn.net/Articles/156274/>). The result should be much improved barrier handling, but it also involves some API changes visible to block drivers. The new code recognizes that different devices will support barriers in different ways. There are three variables which are taken into account: 

  * Whether the device supports ordered tags or not. Ordered tags allows there to be multiple outstanding requests, with the device expected to handle them in the indicated order. In the absence of ordered tags, barriers can only be implemented by stopping the request queue and being sure that requests before the barrier complete before any subsequent requests are issued. 

  * Whether an explicit flush operation is required prior to issuing the barrier operation. Devices which perform write caching usually will need to be flushed for the barrier semantics to be met. 

  * Whether the device supports the "forced unit access" (FUA) mode. If FUA is supported, the actual barrier request can be issued in FUA mode, and there is no need to force a flush afterward. In the absence of FUA, flushes are usually required before and after the barrier operation. 

A block driver will tell the system about how its device operates with `blk_queue_ordered()`, which has a new prototype: 

```
typedef void (prepare_flush_fn)(request_queue_t *q, 
                                        struct request *rq);
        int blk_queue_ordered(request_queue_t *q, unsigned ordered,
    		          prepare_flush_fn *prepare_flush_fn,
    		          unsigned gfp_mask);
```

The `ordered` parameter describes how barriers to be implemented; it has values like `QUEUE_ORDERED_DRAIN_FLUSH` to indicate that barriers are implemented by stopping the queue, and that flushes are required both before and after the barrier; or `QUEUE_ORDERED_TAG`, which says that ordered tags handle everything. The `prepare_flush_fn()` will be called to do whatever is required to make a specific operation force a flush to physical media. See [Tejun's documentation patch](<https://lwn.net/Articles/157209/>) for more details. 

With the above information in hand, the block layer can handle the implementation of barrier requests. As long as the driver implements flushes when requested and recognizes I/O requests requiring the FUA mode (a helper function `blk_fua_rq()` is provided for this purpose), the rest is taken care of at the higher levels. 

The barrier patch also adds an `uptodate` parameter to `end_that_request_last()`. This API change, which will affect most block drivers, is necessary to enable drivers to signal errors for non-filesystem requests. 

The conversation on the lists suggests that both of the above patches are headed for the mainline sooner or later. Mike Christie's [block layer multipath patch](<http://lwn.net/Articles/156058/>) may take a little longer, however. The question of where multipath support should be implemented has often been discussed; more recently, the seeming consensus was that the device mapper layer was the right place. The result was that the [device mapper multipath patches](<http://lwn.net/Articles/124703/>) were merged early this year. So it is a bit surprising to see the issue come back now. 

Mike has a few reasons for wanting to implement multipath at the lower level. These include: 

  * Dealing with multipath hardware involves a number of strange SCSI commands, and, especially, error codes. With the current implementation, it is hard to get detailed error information up to the device mapper layers in any sort of generic way. 

  * Lower-level multipath makes it easier to merge device commands (such as failover requests) with the regular I/O stream. 

  * The request queue mechanism is a better place for handling retries and other related tasks. 

  * Placing the I/O scheduler above the multipath mechanism allows scheduling decisions to be made at the right time. 

  * In theory, a wider range of devices could benefit from the multipath implementation - should anybody have a need for a multipath tape drive. 

A number of code simplifications are also said to result from the new organization. The new multipath code is essentially a repackaging of the device mapper code, reworked to deal with the block layer from underneath. It not being proposed for merging at this time, or even for serious review. So far, there has been little discussion of this patch.
