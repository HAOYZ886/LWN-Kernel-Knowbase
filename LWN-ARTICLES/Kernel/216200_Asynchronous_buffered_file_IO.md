---
title: Asynchronous buffered file I/O
url: https://lwn.net/Articles/216200/
date: "January 3, 2007"
category: Asynchronous IO
author: 
---

Asynchronous I/O (AIO) operations have the property of not blocking in the kernel. If an operation cannot be completed immediately, it is set in motion and control returns to the calling application while things are still in progress. This functionality allows a suitably-programmed application to keep multiple operations going in parallel without blocking on any of them. 

While Linux has long offered a set of system calls for asynchronous I/O, support within the kernel has been spotty and slow in coming. Most char devices do not provide the necessary methods - generally because there is no pressing need for them to support asynchronous operations. Networking supports AIO reasonably well. At the block level, all I/O is asynchronous, but that is not true when dealing with the virtual filesystem layer. Quite a bit of work went into supporting asynchronous direct filesystem I/O, making the big database vendors happy. But most applications do not use direct I/O, and the system as a whole usually benefits from the use of buffered I/O. So asynchronous buffered I/O support is arguably the biggest remaining hole. 

Various buffered filesystem AIO patches have been posted over the course of some three years, but none have made it into the kernel. Recently, Suparna Bhattacharya has restarted this work with [a new file AIO patch](<https://lwn.net/Articles/216202/>) which attempts to add this capability in the least intrusive way possible. This work may now be simple enough that few will be able to find things to object to. 

Like previous versions of the patch, the current code adds a special wait queue to each process's task structure. That queue is used for normal synchronous operations, while asynchronous operations each have their own, dedicated queue. The current wait queue is passed into filesystem I/O operations which could block. That enables a couple of special tricks to be performed: 

  * The I/O wait code checks to see if an asynchronous wait queue is in use. If so, it simply returns `-EIOCBRETRY` rather than waiting. This return code indicates that the operation is still in progress; among other things, it is used to ensure that the wait queue entry remains on the queue until the operation completes. 

  * Normally, wait queues wake up whatever process is waiting on them. They are, however, rather more general than that. By changing the wakeup function (see [this LWN article](<http://lwn.net/Articles/22913/>) for information on how to do that), the AIO code can use wait queues as notification service. When a "wakeup" happens on a queue being used for AIO, the kernel, rather than waking up a process, starts up a workqueue with an entry that will take the next step in the I/O operation. 

The normal buffered filesystem read code, simplified almost into oblivion, looks something like this: 

```
for each file page to be read
    	get the page into the page cache
    	copy the contents to the user buffer
```

The real code can be found in `mm/filemap.c` as `do_generic_mapping_read()`, but the leading comment notes that ""this is really ugly"". It is one of only three functions so marked in that file, so, trust your editor, and go with the simple version above. 

In the pseudocode version, the place where things block is clearly the step where the file page is read into the page cache. If the page is not already cached, the kernel will have to set up a disk I/O operation and wait for it to be carried out. That code proceeds the way it always did, until it gets to the "wait" part, at which point the AIO wait queue will be noticed and the code will return to whatever it was doing before. Once the read completes, the special wakeup function associated with the AIO queue will pick up where things left off. 

One might well wonder just how that "pick up" part works. The wakeup function will not be running in the process of the original calling application, and may well not be running in process context at all. So it queues up a workqueue function which will examine the state of the outstanding I/O operation and, if necessary, jump back into the loop above to continue the work. Before doing so, however, the workqueue function carefully tweaks its memory management context so that it shares the original application's address space. That tweak is necessary to make the final line above (copy the page to the user buffer) work as expected. The workqueue function will perform that copy, then proceed on to the next page (if any). Likely as not, that next page will need to be read in from disk, so the workqueue function will, after ensuring that the operation is started, simply quit. This process repeats until all of the requested data has been read, at which point the application can be notified that the operation is complete. 

On the write side, one might think that no changes are required - buffered file writes are already asynchronous, with the flush to disk happening in the background. The exception, however, is when `O_SYNC` is in use. There are situations where applications want to know when the data has found its way to the disk platter, but they still don't want to block waiting for that to happen. A very similar approach is used to make asynchronous `O_SYNC` writes work, though the patch is a little larger. A couple of the low-level page writeback functions required modifications so that they would pass the relevant wait queue around. 

Even with this change in place, writes can still block on occasion. In particular, any operation which requires allocating disk blocks for the file may block while those allocations are performed. This issue can probably be worked around, but that work has not yet been done. 

The result of all this is a working asynchronous buffered file I/O capability which makes almost no changes to (and adds little overhead to) the "normal" synchronous code. If no serious objections are raised, the Linux AIO subsystem might just become a little more complete in the near future.
