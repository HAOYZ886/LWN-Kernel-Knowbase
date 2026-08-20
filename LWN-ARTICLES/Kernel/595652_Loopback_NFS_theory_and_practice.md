---
title: "Loopback NFS: theory and practice"
url: https://lwn.net/Articles/595652/
date: "April 23, 2014"
category: "Clusters-Filesystems; Filesystems-NFS"
author: "April 23, 2014 This article was contributed by Neil Brown"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

April 23, 2014

This article was contributed by Neil Brown

The Linux NFS developers have [long known](<http://comments.gmane.org/gmane.linux.nfs/8393>) that mounting an NFS filesystem onto the same host that is exporting it (sometimes referred to as a loopback or localhost NFS mount) can lead to deadlocks. Beyond [one patch](<https://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=b91404d3496c7dd004f3e797b03fbf9795e1e7fa>) posted over ten years ago, little effort has been put into resolving the situation as no convincing use case was ever presented. Testing of the NFS implementation can certainly benefit from loopback mounts; this use probably triggered the mentioned patch. With that fix in place, the remaining deadlocks do take some effort to trigger, so the advice to testers was essentially "be careful and you should be safe".

For any other use case, it would seem that using a "bind" mount would provide a result that is indistinguishable from a loopback NFS mount. In short: if it hurts when you use a loopback NFS mount, then simply don't do that. However, a convincing use case recently came to light which motivated more thought on the issue. It led this author on an educational tour of the interaction between filesystems and memory management, and produced a [recently posted patch set](<https://lwn.net/Articles/595743/>) (replacing an [earlier attempt](<http://comments.gmane.org/gmane.linux.kernel.mm/115743>)) which removes most, and hopefully all, such deadlocks.

#### A simple cluster filesystem

That use case involves using NFS as the filesystem in a high-availability cluster where all hosts have shared access to the storage. For all nodes in the cluster to be able to access the storage equally, you need some sort of cluster filesystem like OCFS2, Ceph, or GlusterFS. If the cluster doesn't need particularly high levels of throughput and if the system administrator prefers to stick with known technology, NFS can provide a simple and tempting alternative.

To use NFS as a cluster filesystem, you mount the storage on an arbitrary node using a local filesystem (ext4, XFS, Btrfs, etc), export that filesystem using NFS, then mount the NFS filesystem on all other nodes. The node exporting the filesystem can make it appear in the local namespace in the desired location using bind mounts and no loopback NFS is needed — at least initially.

As this is a high-availability cluster, it must be able to survive the failure of any node, and particularly the failure of the node running the NFS server. When this happens, the cluster-management software can mount the filesystem somewhere else. The new owner of the filesystem can export it via NFS and take over the IP address of the failed host; all nodes will smoothly be able to access the shared storage again. All nodes, that is, except the node which has taken over as the NFS server.

The new NFS-serving node will still have the shared filesystem mounted via NFS and will now be accessing it as a loopback NFS mount. As such, it will be susceptible to all the deadlocks that have led us to recommend against loopback NFS mounts in the past. In this case, it is not possible to "simply use bind mounts" as the filesystem is already mounted, applications are already using it and have files open, etc. Unmounting that filesystem would require stopping those applications — an action which is clearly contrary to the high-availability goal. 

This scenario is clearly useful, and clearly doesn't work. So what was previously a wishlist item, and quite far from the top of the list at that, has now become a bug that needs fixing.

#### Theory meets practice

The deadlocks that this scenario trigger generally involve a sequence of events like: (1) the NFS server tries to allocate memory, (2) the memory allocator then tries to free memory by writing some pages out to the filesystem via the NFS client, and (3) the NFS client waits for the NFS server to make some progress. My assumption had been that this deadlock was inevitable because the same memory manager was trying to serve two separate but competing users: the NFS client and the NFS server.

A possible fix might be to run the NFS server inside a virtual machine, and to give this VM a fixed and locked allocation of memory so there would not be any competition. This would work, but it is hardly the simple solution that our administrator was after and would likely present challenges in sizing the VM for optimal performance.

This is where I might have left things had not a colleague, Ulrich Schairer, presented me with a system which was deadlocking exactly as described and said, effectively, "It's broken, please fix". I reasoned that analyzing the deadlock would at least allow me to find a precise answer as to why it cannot work. I now know it led to more than that. After a sequence of patches and re-tests it turned out that there were two classes of problem, each of which differed in important ways from the problem which was addressed 10 years ago. Trying to understand these problems led to an exploration of the nature and history of the various mechanisms already present in Linux to avoid memory-allocation deadlocks as [reported on](<http://lwn.net/Articles/594725/>) last week.

With that context, it might seem that some manipulation of the `__GFP_FS` and/or `PF_FSTRANS` flags should allow the deadlock to be resolved. If we think of `nfsd` as simply being the lower levels of the NFS filesystem, then the deadlock involves a lower layer of a filesystem allocating memory and thus triggering writeout to that same filesystem. This is exactly the deadlock that `__GFP_FS` was designed to prevent, and, in fact, setting `PF_FSTRANS` in all `nfsd` threads did fix the deadlock that was the easiest to hit.

Further investigation revealed, as it often does, that reality is sometimes more complex than theory might suggest. Using the `__GFP_FS` infrastructure, either directly or through `PF_FSTRANS`, turns out to be neither sufficient, nor desirable, as a solution to the problems with loopback NFS mounts. The remainder of this article explores why it is not sufficient and next week we will conclude with an explanation of why it isn't even desirable.

#### A pivotal patch

Central to understanding both sides of this problem is a [change that happened](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=ee72886d8ed5d9de3fa0ed3b99a7ca7702576a96>) in Linux 3.2. This change was authored by my colleague Mel Gorman who fortunately sits just on the other side of the Internet from me and has greatly helped my understanding of some of these issues (and provided valuable review of early versions of this article). This patch series changed the interplay between memory reclaim and filesystem writeout in a way that, while not actually invalidating `__GFP_FS`, changed its importance.

Prior to 3.2, one of the several strategies that memory reclaim would follow was to initiate writeout of any dirty filesystem pages that it found. Writing a dirty page's contents to persistent storage is an obvious requirement before the page itself can be freed, so it would seem to make sense to do it while looking for pages to free. Unfortunately, it had some serious negative side effects.

One side effect was the amount of kernel stack space that could be used. The `writepage()` function in some filesystems can be quite complex and, as a result, can quite reasonably use a lot of stack space. If a memory allocation request was made in some unrelated code that also used a lot of stack space, then the fact that memory allocation led directly to memory reclaim and, from there, to filesystem writeout, meant that two heavy users of stack space could be joined together, significantly increasing the total amount of stack space that could be required. In some cases, the amount of space needed could exceed the size of the kernel stack.

Another side effect is that pages could be written out in an unpredictable order. Filesystems tend to be optimized to write pages out in the order they appear in the file, first page first. This allows space on the storage device to be allocated optimally and allows multiple pages to be easily grouped into fewer, larger writes. When multiple processes are each trying to reclaim memory, and each is writing out any dirty pages it finds, the result is somewhat less orderly than we might like.

Hence the change in Linux 3.2 removed writeout from direct reclaim, leaving it to be done by `kswapd` or the various filesystem writeback threads. In such a complex system as Linux memory management, a little change like that should be expected to have significant follow-on effects, and the patch mentioned above was really just the first of a short series which made the main change and then made some adjustments to restore balance. The particular adjustment which interests us was to [add a small delay](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=92df3a723f84cdf8133560bbff950a7a99e92bc9>) during reclaim.

#### Waiting for writeout

The writeout code that was removed would normally avoid submitting a write if doing so might block. This can happen if the block I/O request queue is full and the submission needs to wait for a free slot; it can be avoided by checking if the backing device is "congested". However, if the process that is allocating memory is in the middle of writing to a file on a particular device, and the memory reclaim code finds a dirty page that can be written to that same device, then it skips the congestion test and, thus, it may well block. This has the effect of slowing down any process writing to a device to match the speed of the device itself and is important for keeping balance in memory management.

With the change so that direct reclaim would no longer write out dirty file pages, this delay no longer happened (though the `backing_device_info` field of the task structure which enabled the delay is still present with no useful purpose). In its place, we get an explicit small delay if all the dirty pages looked at are waiting for a congested backing device. This delay causes problems for loopback NFS mounts. In contrast to the implicit delay present before Linux 3.2, this delay is not avoided by clearing `__GFP_FS`. This is why using `__GFP_FS` or `PF_FSTRANS` is not sufficient.

Understanding this problem requires an understanding of the "[backing device](<http://lxr.free-electrons.com/source/include/linux/backing-dev.h?v=3.14#L64>)" object, an abstraction within Linux that holds some important information about the storage device underlying a filesystem. This information includes the recommended read-ahead size and the request queue length — and also whether the device is congested or not. For local filesystems `struct backing_dev_info` maps directly to the underlying block device (though, for Btrfs, which can have multiple block devices, there are extra challenges). For NFS, the queue in this structure is a list of requests to be sent to the NFS server rather than to a physical device. When this queue reaches a predefined size, the backing device for the NFS filesystem will be designated as "congested".

If the backing device for a loopback-mounted NFS filesystem ever gets congested while memory is tight, we have a problem. As `nfsd` tries to allocate pages to execute write requests, it will periodically enter reclaim and, as the NFS backing device is congested, it will be forced to sleep for 100ms. This delay will slow NFS throughput down to several kilobytes per second and so will ensure that the NFS backing device remains congested. This does not actually result in a deadlock as forward progress is achieved, but it is a livelock resulting in severely reduced throughput, which is nearly as bad.

This situation is very specific to our NFS scenario, as the problem is caused by a backing device writing into the page cache. It is not really a general filesystem recursion issue, so it is not the same sort of problem that might be addressed with suitable use of `__GFP_FS`.

#### Learning from history

This issue is, however, similar to the problem from ten years ago that was fixed by the patch mentioned in the introduction. In that case, the problem was that a process which was dirtying pages would be slowed down until a matching number of dirty pages had been written out. When this happened, `nfsd` could end up being blocked until `nfsd` had written out some pages, thus producing a deadlock. In our present case, the delay happens when reclaiming memory rather than when dirtying memory, and the delay has an upper limit of 100ms, but otherwise it is a similar problem.

The solution back then was to add a per-process flag called `PF_LESS_THROTTLE`, which was set only for `nfsd` threads. This flag increased the threshold at which the process would be slowed down (or "throttled") and so broke the deadlock. There are two important ideas to be seen in that patch: use a per-process flag, and do not remove the throttling completely but relax it just enough to avoid the deadlock. If `nfsd` were not throttled at all when dirtying pages, that would just cause other problems.

With our 100ms delay, it is easy to add a test for the same per-process flag, but the sense in which the delay should only be partially removed is somewhat less obvious.

The problem occurs when `nfsd` is writing to a local filesystem, but the NFS queue is congested. `nfsd` should probably still be throttled when that local filesystem is congested, but not when the NFS queue is congested. If other queues are congested, it probably doesn't matter very much whether `nfsd` is throttled or not, though there is probably a small preference in favor of throttling.

As the `backing_dev_info` field of the task structure was (fortuitously) not removed when direct-reclaim writeback was removed in 3.2, we can easily use `PF_LESS_THROTTLE` to avoid the delay in cases where `current->backing_dev_info` (i.e. the backing device that `nfsd` is writing to) is not congested. This may not be completely ideal, but it is simple and meets the key requirements, so should be safe ... providing it doesn't upset other users of `PF_LESS_THROTTLE`.

Though `PF_LESS_THROTTLE` has only ever been used in `nfsd`, there have been various patches proposed between [2005](<http://mirror.vtx.ch/patches/downloads/linux/linux-2.6.11.7-loop_AES-3.0c.patch>) and [2013](<https://lkml.org/lkml/2013/10/16/348>) adding the flag to the writeback process used by the `loop` block device, which makes a regular file behave like a block device. This process is in exactly the same situation as `nfsd`: it implements a backing device by writing into the page cache. As such, it can be expected to face exactly the same problems as described above and would equally benefit from having `PF_LESS_THROTTLE` set and having that flag bypass the 100ms delay. It is probably only a matter of time before some patch to add `PF_LESS_THROTTLE` to `loop` devices will be accepted.

There are three other places where direct reclaim can be throttled. The first is the function `throttle_direct_reclaim()`, which was [added](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=5515061d22f0f9976ae7815864bfd22042d36848>) in 3.6 as part of swap-over-NFS support. This throttling is explicitly disabled for any kernel threads (i.e. processes with no user-space component). As both `nfsd` and the `loop` device thread are kernel threads, this function cannot affect users of `PF_LESS_THROTTLE` and so need not concern us.

The other two are in `shrink_inactive_list()` (the same function which holds the [primary source](<http://lxr.free-electrons.com/source/mm/vmscan.c?v=3.14#L1559>) of our present pain). The [first](<http://lxr.free-electrons.com/source/mm/vmscan.c?v=3.14#L1448>) of these repeatedly calls `congestion_wait()` until there aren't too many processes reclaiming memory at the same time, as this can upset some heuristics. This has previously led to a deadlock that was fixed by avoiding the delay whenever `__GFP_FS` or `__GFP_IO` was cleared. Further discussion of this will be left to next time when we examine the use of `__GFP_FS` more closely.

The [last delay](<http://lxr.free-electrons.com/source/mm/vmscan.c?v=3.14#L1548>) is near the end of `shrink_active_list()`; it adds an extra delay (via `congestion_wait()` again) when it appears that the flusher threads are struggling to make progress. While a livelock triggered by this delay has not been seen in testing, it is conceivable that the flusher thread could block when the NFS queue is congested; that could lead to `nfsd` suffering this delay as well and so keeping the queue congested. Avoiding this delay in the same conditions as the other delay seems advisable.

#### One down, one to go

With the livelocks under control, not only for loopback NFS mounts but potentially for the `loop` block device as well, we only need to deal with one remaining deadlock. As we found with this first problem, the actual change required will be rather small. The effort to understand and justify that change, which will be explored next week, will be somewhat more substantial.
