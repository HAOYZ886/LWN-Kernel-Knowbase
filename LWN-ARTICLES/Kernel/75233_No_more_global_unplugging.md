---
title: No more global unplugging
url: https://lwn.net/Articles/75233/
date: "March 10, 2004"
category: "Block layer-Plugging"
author: 
---

The block layer supports the notion of "plugging" a request queue for a block device. A plugged queue passes no requests to the underlying device; it allows them to accumulate, instead, so that the I/O scheduler has a chance to reorder them and optimize performance. There comes a time, however, when the plug has to be pulled and the device restarted. Often, code within the filesystem or virtual memory layers decides that, for whatever reason, it's time to get block I/O moving again. In the current 2.6 kernel, there is a function (`blk_run_queues()`) which performs this task. 

The problem is that `blk_run_queues()` has turned out to be a bit of a performance and scalability problem. It has a single, global lock which keeps multiple processors from trying to restart the queues at the same time; this lock has become a bit of a contention point on some systems. A call to `blk_run_queues()` also restarts all block devices on the system, even though there is typically only one queue that truly needs to be unplugged. 

To address these problems, Jens Axboe has posted [a patch](<https://lwn.net/Articles/75099/>) which does away with `blk_run_queues()` altogether. This change is a result of a fundamental realization: there is always one specific queue which needs to be kickstarted. So `blk_run_queues()` has been replaced with `blk_run_queue()` (which takes the specific queue to start as a parameter) and `blk_run_address_space()` (which takes a pointer to a `address_space` structure). With these functions, higher-level code can fire up the request queue which belongs to a specific device or which ultimately underlies a particular non-anonymous mapping. 

This patch is going straight into the -mm tree; Andrew Morton [commented](<https://lwn.net/Articles/75239/>) ""This is such an improvement over what we have now it isn't funny."" He also noted that ""...the next -mm is starting to look like linux-3.1.0..."" The 2.6 kernel looks to be interesting for a while.
