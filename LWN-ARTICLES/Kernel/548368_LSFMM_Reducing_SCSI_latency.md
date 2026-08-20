---
title: "LSFMM: Reducing SCSI latency"
url: https://lwn.net/Articles/548368/
date: "April 25, 2013"
category: SCSI
author: "By Jonathan Corbet April 25, 2013 LSFMM Summit 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 25, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

As was noted often during the 2013 LSFMM Summit, the speed of storage devices is increasing rapidly, with the result that the Linux storage stack is having a hard time driving those devices at their full speed. For much of that hardware, one of the more significant parts of the stack is the SCSI layer. A session led by Bart van Assche examined ways in which the SCSI code could be made to perform better with fast hardware. 

The discussion quickly honed in on the issue of the SCSI queue depth parameter, which limits the number of outstanding I/O operations. Bart complained that the queue depth should really be a per-LUN (per-device) parameter, rather than per-host; that would allow more outstanding requests and, hopefully, better performance. It could also reduce lock contention since queue depth counter updates could be split across multiple counters. James Bottomley objected that the queue depth limit is already a per-LUN parameter, but that host adapters tend to have a per-host limit as well. In the end, SCSI commands must go through the host adapter regardless of the target LUN, so there will be locking at the host adapter level regardless. 

James went on to say that the queue depth was really the wrong problem to be worried about. Speeding up the SCSI layer requires removing lock contention, and that is best done by going to a multiple-queue architecture. There was talk of setting up one queue per LUN, but James stated that per-LUN queues are the wrong model. The right way to do multiqueue I/O is to have per-CPU queues, because that's the level at which locks can be eliminated — besides, with LUN numbers being 64-bit quantities, one could need a _lot_ of queues. So per-CPU queuing is the plan once Jens Axboe's multiqueue block layer implementation is ready. That code will make it possible to split the SCSI stack up on a per-CPU basis and minimize the interactions between the CPUs. 

Even with a single queue, Jens added, there is a lot that can be done to minimize contention. Much of that work seems to have to do with clever tagging of SCSI commands so that they can be dispatched quickly to the appropriate CPU. True multiqueue hardware will have per-queue tagging, which will make things even easier. 

Bart's final question was: should the SCSI layer move to being a `make_request_fn()` driver? Block drivers that specify their own "make request" function accept I/O requests almost directly from the rest of the kernel, shorting out much of the block layer's functionality. Taking that approach can look like a way to reduce overhead but, as Jens said, it is a model that the block developers are trying to get away from. Using `make_request_fn()` means taking on a lot of the tasks that are otherwise handled in the block layer, leading to duplicated solutions to the same problem. Even if the SCSI layer were to be made more block-like (by using BIO structures throughout the midlayer, for example), there would still be a lot of infrastructure that would need to be reimplemented. 

There was some talk of NUMA systems where I/O devices, too, are local to a specific CPU. In such cases, it obviously makes sense to move the I/O processing work to the right processor. A more NUMA-aware scheduler will help in this regard, but there were concerns that the scheduler still won't know about the system's I/O topology. The system's tendency to move processes toward the CPU where wakeup events occur should help to get the I/O threads in the right place. There might still be value in setting explicit thread CPU affinities on complex systems, though. 

The last part of the session returned to the tagging of SCSI requests in a multiqueue environment. Since tags are a part of the request-completion notification from the device, it would be nice if the value of the tag, itself, could direct processing immediately to the correct queue. Tags are currently 16 bits wide, so including the queue was said to be "vaguely possible"; the T10 committee (which writes the SCSI standard) is considering increasing the width of tags to make the inclusion of queue pointers easier. 

But wider tags may not really be needed. Real-world devices, it seems, do not generally operate with a queue depth greater than 255, so eight bits of the tag value are sufficient to track the requests to any specific device. That leaves eight bits that can be used to encode a queue number. James expressed some relief that upcoming devices did not appear to need queues larger than that; having to deal with massive queues, he said, would be bad for latency. Not having to plan for that case, he said, will make life a little easier. 

[Thanks to Elena Zannoni, whose extensive notes made this writeup possible.]
