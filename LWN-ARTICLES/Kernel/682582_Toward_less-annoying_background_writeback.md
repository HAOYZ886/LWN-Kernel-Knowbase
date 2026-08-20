---
title: "Toward less-annoying background writeback"
url: https://lwn.net/Articles/682582/
date: "April 13, 2016"
category: "Memory management-Writeback"
author: "By Jonathan Corbet April 13, 2016"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 13, 2016

It's an experience many of us have had: write a bunch of data to a relatively slow block device, then try to get some other work done. In many cases, the system will slow to a crawl or even appear to freeze for a while; things do not recover until the bulk of the data has been written to the device. On a system with a lot of memory and a slow I/O device, getting things back to a workable state can take a long time, sometimes measured in minutes. Linux users are understandably unimpressed by this behavior pattern, but it has been stubbornly present for a long time. Now, perhaps, a new patch set will improve the situation. 

That patch set, from block subsystem maintainer Jens Axboe, is titled "[Make background writeback not suck](<https://lwn.net/Articles/681763/>)." "Background writeback" here refers to the act of flushing block data from memory to the underlying storage device. With normal Linux buffered I/O, a `write()` call simply transfers the data to memory; it's up to the memory-management subsystem to, via writeback, push that data to the device behind the scenes. Buffering writes in this manner enables a number of performance enhancements, including allowing multiple operations to be combined and enabling filesystems to improve layout locality on disk. 

So how is it that a performance-enhancing technique occasionally leads to such terrible performance? Jens's diagnosis is that it has to do with the queuing of I/O requests in the block layer. When the memory-management code decides to write a range of dirty data, the result is an I/O request submitted to the block subsystem. That request may spend some time in the I/O scheduler, but it is eventually dispatched to the driver for the destination device. Getting there requires passing through a series of queues. 

The problem is that, if there is a lot of dirty data to write, there may end up being vast numbers (as in thousands) of requests queued for the device. Even a reasonably fast drive can take some time to work through that many requests. If some other activity (clicking a link in a web browser, say, or launching an application) generates I/O requests on the same block device, those requests go to the back of that long queue and may not be serviced for some time. If multiple, synchronous requests are generated — page faults from a newly launched application, for example — each of those requests may, in turn, have to pass through this long queue. That is the point where things appear to just stop. 

In other words, the block layer has a [bufferbloat](<https://lwn.net/Articles/616241/>) problem that mirrors the issues that have been seen in the networking stack. Lengthy queues lead to lengthy delays. 

As with bufferbloat, the answer lies in finding a way to reduce the length of the queues. In the networking stack, techniques like [byte queue limits](<https://lwn.net/Articles/454390/>) and [TCP small queues](<https://lwn.net/Articles/507065/>) have mitigated much of the bufferbloat problem. Jens's patches attempt to do something similar in the block subsystem. 

#### Taming the queues

Like networking, the block subsystem has queuing at multiple layers. Requests start in a submission queue and, perhaps after reordering or merging by an I/O scheduler, make their way to a dispatch queue for the target device. Most block drivers also maintain queues of their own internally. Those lower-level queues can be especially problematic since, by the time a request gets there, it is no longer subject to the I/O scheduler's control (if there is an I/O scheduler at all). 

Jens's patch set aims to reduce the amount of data "in flight" through all of those queues by throttling requests when they are first submitted. To put it simply, each device has a maximum number of buffered-write requests that can be outstanding at any given time. If an incoming request would cause that limit to be exceeded, the process submitting the request will block until the length of the queue drops below the limit. That way, other requests will never be forced to wait for a long queue to drain before being acted upon. 

In the real world, of course, things are not quite so simple. Writeback is not just important for ensuring that data makes it to persistent storage (though that is certainly important enough); it is also a key activity for the memory-management subsystem. Writeback is how dirty pages are made clean and, thus, available for reclaim and reuse; if writeback is impeded too much, the system could find itself in an out-of-memory situation. Running out of memory can lead to other user-disgruntling delays, along with unleashing the OOM killer. So any writeback throttling must be sure to not throttle things too much. 

The patch set tries to avoid such unpleasantness by tracking the reason behind each buffered-write operation. If the memory-management subsystem is just pushing dirty pages out to disk as part of the regular task of making their contents persistent, the queue limit applies. If, instead, pages are being written to make them free for reclaim — if the system is running short of memory, in other words — the limit is increased. A higher limit also applies if a process is known to be waiting for writeback to complete (as might be the case for an `fsync()` call). On the other hand, if there have been any non-writeback requests within the last 100ms, the limit is reduced below the default for normal writeback requests. 

There is also a potential trap in the form of drives that do their own write caching. Such drives will indicate that a write request has completed once the data has been transferred, but that data may just be sitting in a cache within the drive itself. In other words, the drive, too, may be maintaining a long queue. In an attempt to avoid overfilling that queue, the block layer will impose a delay between write operations on drives that are known to do caching. That delay is 10ms by default, but can be tweaked via a sysfs knob. 

Jens tested this work by having one process write 100MB each to 50 files while another process tries to read a file. The reading process will, on current kernels, be penalized by having each successive read request placed at the end of a long queue created by all those write requests; as might be expected, it performs poorly. With the patches applied, the writing processes take a little longer to complete, but the reader runs much more quickly, with far fewer requests taking an inordinately long period of time. 

This is an early-stage patch set; it is not expected to go upstream in the near future. Patches that change memory-management behavior can often cause unexpected problems with different workloads, so it takes a while to build confidence in a significant change, even after the development work is deemed to be complete (which is not the case here). Indeed, Dave Chinner has already [reported](<https://lwn.net/Articles/683353/>) a performance regression with one of his testing workloads. The tuning of the queue-size limits also needs to be made automatic if possible. There is clearly work still to be done here; the patch set is also likely to be a subject of discussion at the upcoming [Linux Storage, Filesystem, and Memory-Management Summit](<http://events.linuxfoundation.org/events/linux-storage-filesystem-and-mm-summit>). So users will have to wait a bit longer for this particular annoyance to be addressed.
