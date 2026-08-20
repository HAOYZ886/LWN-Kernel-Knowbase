---
title: Flushing out pdflush
url: https://lwn.net/Articles/326552/
date: "April 1, 2009"
category: "Block layer; Memory management-Writeback"
author: "April 1, 2009 This article was contributed by Goldwyn Rodrigues"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

April 1, 2009

This article was contributed by Goldwyn Rodrigues

The kernel page cache contains in-memory copies of data blocks belonging to files kept in persistent storage. Pages which are written to by a processor, but not yet written to disk, are accumulated in cache and are known as "dirty" pages. The amount of dirty memory is listed in `/proc/meminfo`. Pages in the cache are flushed to disk after an interval of 30 seconds. Pdflush is a set of kernel threads which are responsible for writing the dirty pages to disk, either explicitly in response to a `sync()` call, or implicitly in cases when the page cache runs out of pages, if the pages have been in memory for too long, or there are too many dirty pages in the page cache (as specified by `/proc/sys/vm/dirty_ratio`). 

At a given point of time, there are between two and eight pdflush threads running in the system. The number of pdflush threads is determined by the load on the page cache; new pdflush threads are spawned if none of the existing pdflush threads have been idle for more than one second, and there is more work in the pdflush work queue. On the other hand, if the last active pdflush thread has been asleep for more than one second, one thread is terminated. Termination of threads happens until only a minimum number of pdflush threads remain. The current number of running pdflush threads is reflected by `/proc/sys/vm/nr_pdflush_threads`. 

A number of pdflush-related issues have come to light over time. Pdflush threads are common to all block devices, but it is thought that they would perform better if they concentrated on a single disk spindle. Contention between pdflush threads is avoided through the use of the `BDI_pdflush` flag on the `backing_dev_info` structure, but this interlock can also limit writeback performance. Another issue with pdflush is request starvation. There is a fixed number of I/O requests available for each queue in the system. If the limit is exceeded, any application requesting I/O will block waiting for a new slot. Since pdflush works on several queues, it cannot block on a single queue. So, it sets the `wbc->nonblocking` writeback information flag. If other applications continue to write on the device, pdflush will not succeed in allocating request slots. This may lead to starvation of access to the queue, if pdflush repeatedly finds the queue congested. 

Jens Axboe in his [patch set](<http://lwn.net/Articles/324833/>) proposes a new idea of using flusher threads per backing device info (BDI), as a replacement for pdflush threads. Unlike pdflush threads, per-BDI flusher threads focus on a single disk spindle. With per-BDI flushing, when the request_queue is congested, blocking happens on request allocation, avoiding request starvation and providing better fairness. 

With pdflush, The dirty inode list is stored by the super block of the filesystem. Since the per-BDI flusher needs to be aware of the dirty pages to be written by its assigned device, this list is now stored by the BDI. Calls to flush dirty inodes on the superblock result in flushing the inodes from the list of dirty inodes on the backing device for all devices listed for the filesystem. 

As with pdflush, per-BDI writeback is controlled through the `writeback_control` data structure, which instructs the writeback code what to do, and how to perform the writeback. The important fields of this structure are: 

  * `sync_mode`: defines the way synchronization should be performed with respect to inode locking. If set to WB_SYNC_NONE, the writeback will skip locked inodes, where as if set to WB_SYNC_ALL will wait for locked inodes to be unlocked to perform the writeback. 

  * `nr_to_write`: the number of pages to write. This value is decremented as the pages are written. 

  * `older_than_this`: If not NULL, all inodes older than the jiffies recorded in this field are flushed. This field takes precedence over `nr_to_write`. 

The struct bdi_writeback keeps all information required for flushing the dirty pages: 

```
struct bdi_writeback {
    	struct backing_dev_info *bdi;
    	unsigned int nr;
    	struct task_struct	*task;
    	wait_queue_head_t	wait;
    	struct list_head	b_dirty;
    	struct list_head	b_io;
    	struct list_head	b_more_io;
    
    	unsigned long		nr_pages;
    	struct super_block	*sb;
        };
```

The `bdi_writeback` structure is initialized when the device is registered through `bdi_register()`. The fields of the bdi_writeback are: 

  * `bdi`: the backing_device_info associated with this bdi_writeback, 

  * `task`: contains the pointer to the default flusher thread which is responsible for spawning threads for performing the flushing work, 

  * `wait`: a wait queue for synchronizing with the flusher threads, 

  * `b_dirty`: list of all the dirty inodes on this BDI to be flushed, 

  * `b_io`: inodes parked for I/O, 

  * `b_more_io`: more inodes parked for I/O; all inodes queued for flushing are inserted in this list, before being moved to `b_io`, 

  * `nr_pages`: total number of pages to be flushed, and 

  * `sb`: the pointer to the superblock of the filesystem which resides on this BDI. 

`nr_pages` and `sb` are parameters passed asynchronously to the the BDI flush thread, and are not fixed through the life of the `bdi_writeback`. This is done to facilitate devices with multiple filesystem, hence multiple super_blocks. With multiple super_blocks on a single device, a sync can be requested for a single filesystem on the device. 

The `bdi_writeback_task()` function waits for the `dirty_writeback_interval`, which by default is 5 seconds, and initiates `wb_do_writeback(wb)` periodically. If there are no pages written for five minutes, the flusher thread exits (with a grace period of `dirty_writeback_interval`). If a writeback work is later required (after exit), new flusher threads are spawned by the default writeback thread. 

Writeback flushes are done in two ways: 

  * pdflush style: This is initiated in response to an explicit writeback request, for example syncing inode pages of a super_block. `wb_start_writeback()` is called with the superblock information and the number of pages to be flushed. The function tries to acquire the `bdi_writeback` structure associated with the BDI. If successful, it stores the superblock pointer and the number of pages to be flushed in the `bdi_writeback` structure and wakes up the flusher thread to perform the actual writeout for the superblock. This is different from how pdflush performs writeouts: pdflush attempts to grab the device from the writeout path, blocking the writeouts from other processes. 

  * kupdated style: If there is no explicit writeback requests, the thread wakes up periodically to flush dirty data. The first time one of the inode's pages stored in the BDI is dirtied, the dirtying-time is recorded in the inode's address space. The periodic writeback code walks through the superblock's inode list, writing back dirty pages of the inodes older than a specified point in time. This is run once per `dirty_writeback_interval`, which defaults to five seconds. 

After review of the [first attempt](<http://lwn.net/Articles/322920/>), Jens added functionality of having multiple flusher threads per device based on the suggestions of Andrew Morton. Dave Chinner suggested that filesystems would like to have a flusher thread per allocation group. In the patch set (second iteration) which followed, Jens added a new interface in the superblock to return the `bdi_writeback` structure associated with the inode: 

```
struct bdi_writeback *(*inode_get_wb) (struct inode *);
```

If `inode_get_wb` is NULL, the default `bdi_writeback` of the BDI is returned, which means there is only one `bdi_writeback` thread for the BDI. The maximum number of threads that can be started per BDI is 32\. 

Initial experiments conducted by Jens found an 8% increase in performance on a simple SATA drive running [Flexible File System Benchmark (ffsb)](<http://sourceforge.net/projects/ffsb/>). File layout was smoother as compared to the vanilla kernel as reported by `vmstat`, with a uniform distribution of buffers written out. With a ten-disk btrfs filesystem, per-BDI flushing performed 25% faster. The writeback is tracked by Jens's block layer git tree (git://git.kernel.dk/linux-2.6-block.git) under the "writeback" branch. There have been no comments on the second iteration so far, but per-BDI flusher threads is still not ready enough to go into the 2.6.30 tree. 

Acknowledgments: Thanks to Jens Axboe for reviewing and explaining certain aspects of the patch set.
