---
title: The end of block barriers
url: https://lwn.net/Articles/400541/
date: "August 18, 2010"
category: Block layer; Write barriers
author: "By Jonathan Corbet August 18, 2010"
---

By **Jonathan Corbet**  
August 18, 2010

One of the higher-profile decisions made at the recently-concluded [Linux Storage and Filesystem summit](<http://lwn.net/Articles/399148/>) was to get rid of support for barriers in the Linux kernel block subsystem. This was a popular decision, but also somewhat misunderstood (arguably, by your editor above all). Now, [a new patch series](<http://lwn.net/Articles/399715/>) from Tejun Heo shows how request ordering will likely be handled between filesystems and the block layer in the future. 

The block layer must be able to reorder disk I/O operations if it is to obtain the sort of performance that users expect from their systems. On rotating media, there is much to be gained by minimizing head seeks, and that goal is best achieved by executing all nearby requests together, regardless of the order in which those requests were issued. Even with flash-based devices, there is some benefit to be had by grouping adjacent requests, especially when small requests can be coalesced into larger operations. Proper dispatch of requests to the low-level device driver is normally the I/O scheduler's job; the scheduler will freely reorder requests, blissfully ignorant of the higher-level decisions which created those requests in the first place. 

Note that this reordering also usually happens within the storage device itself; requests will be cached in (possibly volatile) memory and writes will be executed at a time which the hardware deems to be convenient. This reordering is typically invisible to the operating system. 

The problem, of course, is that it is not always safe to reorder I/O requests in arbitrary ways. The classic example is that of a journaling filesystem, which operates in roughly this way: 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

  1. Begin a new transaction. 
  2. Write all planned metadata changes to the journal. Depending on the filesystem and its configuration, data changes may go to the journal as well. 
  3. Write a commit record closing out the transaction. 
  4. Begin the process of writing the journaled changes to the filesystem itself. 
  5. Goto 1. 

If the system were to crash before step 3 completes, everything written to the journal would be lost, but the integrity of the filesystem would be unharmed. If the system crashes after step 3, but before the changes are written to the filesystem, those changes will be replayed at the next mount, preserving both the metadata and the filesystem's integrity. Thus, journaling makes a filesystem relatively crash-proof. 

But imagine what can happen if requests are reordered. If the commit record is written before all of the other changes have been written to the journal, then, after a crash, an incomplete journal would be replayed, corrupting the filesystem. Or, if a transaction frees some disk blocks which are subsequently reused elsewhere in the filesystem, and the reused blocks are written before the transaction which freed them is committed, a crash at the wrong time would, once again, corrupt things. So, clearly, the filesystem must be able to impose some ordering on how requests are executed; otherwise, its attempts to guarantee filesystem integrity in all situations may well be for nothing. 

For some years, the answer has been barrier requests. When the filesystem issues a request to the block layer, it can mark that request as a barrier, indicating that the block layer should execute all requests issued before the barrier prior to doing any requests issued afterward. Barriers should, thus, ensure that operations make it to the media in the right order while not overly constraining the block layer's ability to reorder requests between the barriers. 

In practice, barriers have an unpleasant reputation for killing block I/O performance, to the point that administrators are often tempted to turn them off and take their risks. While the tagged queue operations provided by contemporary hardware should implement barriers reasonably well, attempts to make use of those features have generally run into difficulties. So, in the real world, barriers are implemented by simply draining the I/O request queue prior to issuing the barrier operation, with some flush operations thrown in to get the hardware to actually commit the data to persistent media. Queue-drain operations will stall the device and kill the parallelism needed for full performance; it's not surprising that the use of barriers can be painful. 

In their discussions of this problem, the storage and filesystem developers have realized that the ordering semantics provided by block-layer barriers are much stronger than necessary. Filesystems need to ensure that certain requests are executed in a specific order, and they need to ensure that specific requests have made it to the physical media before starting others. Beyond that, though, filesystems need not concern themselves with the ordering for most other requests, so the use of barriers constrains the block layer more than is required. In general, it was concluded, filesystems should concern themselves with ordering, since that's where the information is, and not dump that problem into the block layer. 

To implement this reasoning, Tejun's patch gets rid of hard-barrier operations in the block layer; any filesystem trying to use them will get a cheery `EOPNOTSUPP` error for its pains. A filesystem which wants operations to happen in a specific order will simply need to issue them in the proper order, waiting for completion when necessary. The block layer can then reorder requests at will. 

What the block layer cannot do, though, is evade the responsibility for getting important requests to the physical media when the filesystem requires it. So, while barrier requests are going away, "flush requests" will replace them. On suitably-capable devices, a flush request can have two separate requirements: (1) the write cache must be flushed before beginning the operation, and (2) the data associated with the flush request itself must be committed to persistent media by the time the request completes. The second part is often called a "force unit access" (or FUA) request. 

In this world, a journaling filesystem can issue all of the journal writes for a given transaction, then wait for them to complete. At that point, it knows that the writes have made it to the device, but the device might have cached those requests internally. The write of the commit record can then follow, with both the "flush" and "FUA" bits set; that will ensure that all of the journal data makes it to physical media before the commit record does, and that the commit record itself is written by the time the request completes. Meanwhile, all other I/O operations - playing through previous transactions or those with no transaction at all - can be in flight at the same time, avoiding the queue stall which characterizes the barrier operations implemented by current kernels. 

The patch set has been well received, but there is still work to be done, especially with regard to converting filesystems to the new way of doing things. Christoph Hellwig has posted [a set of patches](<http://lwn.net/Articles/400777/>) to that end. A lot of testing will be required as well; there is little desire to introduce bugs in this area, since the consequences of failure are so high. But the development cycle has just begun, leaving a fair amount of time to shake down this work before the 2.6.37 merge window opens.
