---
title: Understanding __GFP_FS
url: https://lwn.net/Articles/596618/
date: "April 30, 2014"
category: "Memory management-GFP flags; gfp t"
author: "April 30, 2014 This article was contributed by Neil Brown"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

April 30, 2014

This article was contributed by Neil Brown

As we discovered [last week](<https://lwn.net/Articles/595652/>), using NFS to mount a filesystem onto the same host that is exporting the files has a valuable use case, but is susceptible to deadlocks. These deadlocks involve the `nfsd` server process allocating memory, the memory allocation code choosing some memory pages in the NFS filesystem to free, and the NFS filesystem waiting for a writeout to complete — which, of course, needs that `nfsd` server process to make some progress.

As noted, these deadlocks are similar to the situations that triggered the introduction of the `__GFP_FS` memory allocation flag and later the `PF_FSTRANS` process flag. Avoiding the former, or setting the latter, tells the allocation code to never risk waiting for any filesystem operation, so setting `PF_FSTRANS` for `nfsd` threads should remove the deadlocks. It isn't quite that easy, though.

In most cases, separate filesystems are quite separate. Locks held while working on one filesystem will never conflict with locks needed for another filesystem. So when code for, say, XFS needs to allocate memory, the decision whether to set `__GFP_FS` or not can be made entirely within the context of XFS. There are a fixed number of entry points from the memory reclaim code into XFS, and XFS "knows" which locks those entry points might need. If none of those locks are held, then `__GFP_FS` can safely be set to allow calling back into the filesystem if necessary, even if some other lock is held.

#### Multi-lock deadlocks

When a lock outside of a filesystem is taken during memory reclaim, it can have hard-to-foresee consequences. Just such a lock was [introduced](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=35cd78156c499ef83f60605e4643d5a98fef14fd>) in Linux 2.6.32. The purpose of this lock was to limit the number of processes that would be trying to reclaim memory at any one time, so it behaved to some extent like a [counting semaphore](<http://en.wikipedia.org/wiki/Semaphore_%28programming%29>). Rather than counting processes, it actually counts pages of memory that have been taken off some LRU (least recently used) list to be considered for freeing. The effect is much the same, particularly as each process normally examines 32 pages (`SWAP_CLUSTER_MAX`) at a time. If too many pages have been taken off the list (described in the code as being "isolated"), then any new process entering reclaim will have to wait until the number of isolated pages drops below a threshold. 

This delay is the one we postponed discussion of last week. As it is being used to implement a semaphore, we can see it has a different role than the other delays we considered then and is not likely to cause a general slow down of NFS traffic. There is a different kind of hazard associated with this delay mechanism, though.

If the maximum allowed number of pages have been isolated by processes that are all performing reclaim with the `__GFP_FS` flag set (allowing access into filesystems), those processes could all end up blocking on some filesystem lock. If the process holding this lock also tried to reclaim some memory to fulfill an allocation request, it would naturally use `__GFP_FS`. While this would stop it from entering any filesystem, it would not help with the counting semaphore. The process holding the filesystem lock would block on the semaphore which, itself, was held by various processes blocking on the filesystem lock. Thus we get a two-lock deadlock.

This problem is somewhat akin to priority inversion if we think of the `GFP` flag as indicating a priority: an allocation with `__GFP_FS` has a lower priority than an allocation without the flag. As with priority inversion, one resolution could be to increase the priority of any process while it holds the contentious lock. In the present case that would be a terrible solution as it would mean that every process entering reclaim would need to clear `__GFP_FS` while holding the semaphore, and so no reclaimer could ever call into any filesystem.

The [actual fix](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=3cf23841b4b76eb94d3f8d0fb3627690e4431413>) that was applied in Linux 3.8 (3 years later) was to restrict `__GFP_FS` processes to just a small subset (about one-eighth) of the allowed processes; other slots are reserved for processes without that flag. This is a simple strategy which works well with a counting semaphore. With a binary mutex it wouldn't work, as one-eighth of one process doesn't get very far.

#### Finding those deadlocks

When I tried setting `PF_FSTRANS` for `nfsd` threads (to disable `__GFP_FS`), I hit a similar set of problems. Where previously a filesystem only had to be careful about locks that filesystem might take, we now had multiple players that could interact with each other. The local filesystem, the `nfsd` thread, the NFS filesystem and the networking layer in between all have their own specific locking needs and aren't designed to be careful of each other's locks.

Some conflicts were found through testing, but testing is unlikely to find all possible conflicts. The counting semaphore problem was not fixed for three years simply because it took that long to find. Careful examination of the code might find more but, with many filesystems to examine and with the kernel under constant change, it would probably take even more than three years to be confident.

Fortunately Linux has a clever locking validator known as [lockdep](<https://lwn.net/Articles/185666/>) that is good at finding these transitive locking issues. Since [Linux 2.6.30](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=cf40bd16fdad42c053040bcd3988f5fdedbb6c57>), this validator has known about memory-reclaim context, and will report a possible deadlock when the memory allocator takes a lock with `__GFP_FS` set if that lock can also be held while a memory allocation request is made. Had the code which limits the number of processes in reclaim been annotated so that lockdep knew it was a semaphore, the potential deadlock would very likely have been detected much more quickly. This was never done, probably because this code doesn't really _look_ like it is implementing a semaphore.

To enable lockdep validation for `nfsd` is it ([almost](<http://permalink.gmane.org/gmane.linux.kernel.mm/115745>)) as simple as calling

```
lockdep_set_current_reclaim_state(GFP_KERNEL);
```

at the same time that `PF_FSTRANS` is set. This tells lockdep that `nfsd` is part of the memory reclaim process and other reclaimers could be waiting for it. Adding this setting easily triggered a number of warnings. Interpreting the results is not entirely straightforward and there is a possibility of occasional false positives. The overall message quickly became clear though: there are really quite a few locks which are held over a memory allocation and which are taken by `nfsd`, either directly or via some call into a filesystem.

Modifying all these allocations to avoid `__GFP_FS` or changing all the locking calls to set `PF_FSTRANS` while the lock is held would be possible, but that idea would not be ([and was not](<http://permalink.gmane.org/gmane.linux.nfs/62273>)) popular. It would risk upsetting the balance in memory usage, cause performance problems, and in general it involves more change than is really justified to support one narrow feature. So while working with `__GFP_FS` could well lead to a solution, it is not a desirable solution. Fortunately there is another option, as Dave Chinner [helped me to see](<http://permalink.gmane.org/gmane.linux.nfs/62317>).

#### An alternative approach

As mentioned last week, the dynamics of `__GFP_FS` changed significantly in Linux 3.2, when direct memory reclaim stopped writing dirty pages out through filesystems. As controlling this writeout was the original purpose of `__GFP_FS`, it is reasonable to wonder if it is still filling any role -- or, more specifically: what role does it now fill?

If we examine all the places in the kernel where `__GFP_FS` is used, we find that they fall in to three general classes. The first of these is to allow processes to bypass some restrictions in the memory allocator that do not apply to them; the semaphore example mentioned above is one of those. 

More importantly, the second class of use is found in the various object caches in the kernel that support the [shrinker](<https://lwn.net/Articles/550463/>) interface. Shrinkers allow the memory-management subsystem to ask that objects be removed from caches (and their memory freed) when memory gets tight. Shrinker-enabled caches are passed the `GFP` flags when asked to free some objects, and some of them decline if `__GFP_FS` is set, presumably because they might need a lock that could already be held.

To understand if shrinker-related deadlocks might appear when loopback NFS mounts are in use, we need to understand the cases where the NFS client might, under reclaim, block and wait for `nfsd`. The caches which might affect NFS are the generic icache and dcache (which store inodes and dentries — directory entries) and the NFS "access cache," which records the access permissions for various local users (NFS is not allowed to interpret the mode bits or access control list (ACL) directly, but must ask the server about each different user). Careful inspection shows that none of these three caches ever contact the NFS server while freeing items, and they take relatively few locks, none of which are held by other code that might call the NFS server. So, if an allocation from `nfsd` with `__GFP_FS` set ever calls a shrinker for one of these caches, there is no risk of a deadlock.

That just leaves our third class of `__GFP_FS` use that might be relevant for loopback NFS, and that is the `releasepage()` method. Not all `releasepage()` functions care about `__GFP_FS`, but a few do, specifically `nfs_release_page()`.

#### Working with `releasepage()`

The `releasepage()` method (found in `struct address_space_operations`) is provided by filesystems that need to attach extra information to each page in the page cache. Often this is just some per-page metadata that can easily be detached and discarded. `releasepage()` is called when memory allocation code wants to free a given page and reuse it; this call allows that extra information to be freed first.

The NFS filesystem attaches something more substantial to each page. When a page of data is written to the NFS server, it is generally just stored in memory on the server to be written out later. This allows the NFS `WRITE` request to return quickly, and improves throughput. If the server were to crash before the data is written to stable storage, though, the data would be lost, so the client must hold on to the data until it subsequently sends a `COMMIT` request and gets a successful reply.

The extra information that NFS attaches to each page effectively says "I haven't sent a `COMMIT` yet"; `nfs_release_page()` has the job of sending that `COMMIT`, waiting for a reply, and then releasing the page. This can clearly trigger a deadlock if (1) `nfsd` is waiting for an allocation, (2) that allocation is trying to free a page by calling `nfs_release_page()`, and (3) `nfs_release_page()` is waiting for a reply from `nfsd`. Like shrinkers, `releasepage()` is passed a set of GFP flags; the NFS `releasepage()` implementation honors the absence of `__GFP_FS` by refusing to wait for any `COMMIT` operations to complete. So it appears that the only real effect of setting `PF_FSTRANS` in `nfsd` threads is to tell `nfs_release_page()` not to wait for a `COMMIT` to complete. There is nowhere else that NFS is likely to block during allocation: if there were, we would expect to see some `__GFP_FS` related protection.

Simply changing `nfs_release_page()` to not wait at all would avoid the deadlock, but that change would, without doubt, cause other problems. As we saw last week, there can be real value in slowing down the memory reclaim process when freeing memory takes a little while, even to the extent of inserting explicit delays. For similar reasons, waiting in `nfs_release_page()` is generally a good idea. The only problem is in waiting indefinitely. Once the problem has been phrased this way the answer almost falls out. If we replace the indefinite wait in `nfs_release_page()` (or more accurately in the `nfs_commit_inode()` function it calls) with a timed-out wait, then the problem would disappear but the benefits of waiting would remain.

Choosing the ideal timeout requires a bit of guesswork, though; as deadlocks don't happen in practice all that often, a fairly long timeout (maybe even a few seconds) would probably be acceptable. Varying the timeout depending on whether the `COMMIT` request was sent out via an external interface or was routed internally could also help us get the best of both worlds. This, together with the fact that the `wait_on_bit()` interface in the kernel does not currently support timeouts, are just minor details which are easily understood and nearly as easily resolved.

What is important is that the entire problem of lockups with loopback NFS has boiled down to reducing two timeouts. Last-week we found one or two 100ms timeouts that need to be reduced to zero for `nfsd`. This week we found an "infinite timeout" that needed to be reduced to a small value for NFS. It seems to show a certain level of robustness in the (evolving) design of the Linux kernel that such a complex problem would have such a simple solution.

#### Practice meets theory

But there is one small part of the whole puzzle that remains unexplained. Back when first faced with this problem, I assumed that a deadlock would be inevitable. Clearly I was wrong, as I cannot fault the logic that led to a solution. But that sort of reasoning rarely satisfies. Out of all those details, I needed to extract a big-picture understanding of why a deadlock is not inevitable.

The best way to reason about this problem is play the taxonomy game and observe that, with respect to automatic freeing of memory, there are five sorts of memory allocations in the kernel:

  * "Fixed allocations" have a wide range of uses, including holding kernel code and various fixed-sized data structures. As they are never freed (without external intervention), they don't concern us.

  * At the other end of the scale are "cache allocations", which store data that could be read from elsewhere. These include file pages that have not been dirtied and the inode and dentry caches. Allocations of this type can be freed easily (providing we don't try to free them in the wrong context and trigger a deadlock) and so, again, don't concern us.

  * "Transient allocations" are usually small; they will be used for a specific purpose and then freed. A simple example is the `bio` structure which carries data through the block layer and out to disk. To free these, you need only wait.

  * "Dirty file pages" we met briefly last week. When any change is made to a file, a dirty file page results. The total number of dirty file pages is limited by default to a percentage of available memory (the specific percentage has varied over time), though `nfsd` gets a little bonus thanks to `PF_LESS_THROTTLE`.

  * "Dirty anonymous pages" store the data and stack areas for running processes as well as data in `tmpfs` filesystems. There are two ways that this memory can be freed: it can be written out to a swap partition or file, or the owning process can be killed by the out-of-memory killer.

An important requirement when writing pages out to swap is that any memory allocation that might be required must be a "transient allocation" and must either be preallocated (typically using a [mempool](<https://lwn.net/Articles/22909/>)) or must be allocated with `__GFP_MEMALLOC` (described in [part 1](<https://lwn.net/Articles/594725/>)) so that the emergency reserves can be used. Any `__GFP_MEMALLOC` allocation that isn't used for swap-out and isn't a transient allocation is probably a misuse of the flag.

With this understanding, the rules for avoiding deadlocks during memory reclaim are simple:

  1. When writing out a page, make only "transient" allocations.
  2. When writing out an anonymous page, the allocations must come from a `mempool` or from `__GFP_MEMALLOC`.
  3. When writing a file page, it is safe to wait for anonymous pages to be written out, but it is not safe to wait for file pages. So `__GFP_FS` must be absent or disabled in such cases.

If these rules are followed, then any memory that can be freed will be freed. Any allocations required to write to swap will come from the `MEMALLOC` reserves and any memory required to write to a file can come from the 60% or more that doesn't contain dirty file data.

Of course, if we tried to support swap-over-NFS over a loopback mount, things might get a little more … interesting. But that isn't required for the current use case.

#### Implications for `__GFP_FS`

As I try to distill the important lessons to be learned from this exercise, I find that the deadlocks themselves, which first motivated the exercise, are little more than annoying distractions. 

There are two important ideas in managing memory reclaim. One is to slow down or "throttle" processes that are allocating memory, so they don't allocate memory faster than it can be freed and so that all processes can get approximately equal access to memory that does become available. The second is to balance the amount of work done by the process that is allocating memory against the amount of work done by dedicated service processes. Performing that work in the allocating process (i.e. direct reclaim) avoids scheduler overhead, but creates contention between the multiple reclaiming threads and limits the amount of work that can be done as a unit. 

Allowing direct reclaim to block on arbitrary locks tends to conflate these two ideas, as the lock serves to both manage contention and to insert delays. Keeping these two ideas separate would involve having separate explicit delays and using locking primitives, such as `mutex_trylock()`, which never block but which can fail. The use of "trylock" locking would ensure that we minimize scheduler activity and would directly allow the reclaimer to do the easy work, while leaving the harder stuff to dedicated service processes like `kswapd`. 

The use of explicit delays would follow the model displayed in the patch we introduced last week; this approach was also used effectively that same year in the "[No-I/O dirty throttling](<https://lwn.net/Articles/456904/>)" changes that addressed a related problem. It is in the choice of these delays that `__GFP_FS` has an important role. Throttling a process while it holds locks can easily have a multiplying effect by slowing down lots of processes waiting on that lock. By declaring "I'm holding a filesystem lock", `__GFP_FS` indicates that a substantial reduction in any delay would be appropriate. 

Though its original reason for existence was to avoid deadlocks, it seems that the essence of `__GFP_FS` is really about process priority. Deadlocks can be worked around in other ways, as we did for the deadlock created in `nfs_release_page()` (essentially by introducing something a little bit like a trylock operation), but process priority needs to be managed explicitly. The role played by `__GFP_FS` is to make part of that priority management explicit.
