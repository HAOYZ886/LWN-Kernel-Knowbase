---
title: Parallel pathname lookups and the importance of testing
url: https://lwn.net/Articles/692546/
date: "June 29, 2016"
category: "Filesystems-Virtual filesystem layer"
author: "June 29, 2016 This article was contributed by Neil Brown"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

June 29, 2016

This article was contributed by Neil Brown

Parallel pathname lookup is a new development that aims to improve some aspects of Linux filesystem performance. It was discussed at the [2016 Linux Storage, Filesystem, and Memory-Management Summit](<https://lwn.net/Articles/lsfmm2016/>) and, as [we reported at the time](<https://lwn.net/Articles/685108/>), it required two key changes, both of which have subtle consequences making them worthy of closer examination. 

The first of those changes was to introduce a new state for entries in the directory cache (dcache). As well as being positive ("this name does exist") or negative ("this name doesn't currently exist"), they can now be "don't know" or "in-lookup" as it is described in the code. If a dentry (dcache entry) is ever found in this state, the filesystem lookup is still in progress and the caller must wait for it to complete. The design of this change favored performance over simplicity, and the resulting complexity makes bugs harder to see. 

The second change was to replace the per-directory mutex with a read/write semaphore that allows read operations to proceed in parallel. While simple in principle, this change has had performance implications that can be educational. 

As has been [described previously](<https://lwn.net/Articles/649729/>), the dcache allows lookups for cached pathnames to proceed quickly, often with only RCU protection that already allows a high degree of parallelism. The recent work doesn't change this but, instead, handles the case where components of a pathname are _not_ present in the cache. Prior to Linux 4.7-rc1, a per-directory mutex would be held while looking up any name in that directory. For a small directory in a local filesystem, this forced serialization is unlikely to be a problem; looking up one file name is likely to bring the directory block containing the name into the page cache, from which subsequent lookups can be performed with no further delay. For large directories, or directories on network-attached filesystems, it is more likely that every directory access will incur a non-trivial latency and the serialization imposed by the mutex can hurt.

While parallel lookups within a single directory make sense, parallel lookups of a single name do not. Thus, the two changes mentioned can be described as adding per-name locking, and then removing per-directory locking, for lookups at least. The "don't know" state for a dentry could also be described as a "locked" dentry.

The idea of a cache lookup returning an incomplete (but locked) object is far from new. It was in 2002 that [Linux 2.6.12 gained the `iget_locked()` interface](<http://git.kernel.org/cgit/linux/kernel/git/history/history.git/commit/?id=85b640c51ed9e8ba0ab15151d89a98e1a670d347>) that allows the reading of an inode from disk to be separated from the task of adding the inode to the icache (inode cache). At a coarse level, what we are now seeing is the same improvement being added to the dcache. Looking up names in the dcache happens far more frequently than looking up inodes in the icache, so, given that hotter paths tend to be more heavily optimized, it shouldn't be surprising that the dcache version is not as straightforward as [the icache version](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/inode.c?id=2dcd0af568b0cf583645c8a317dd12e344b1c72a#n1073>).

#### A "don't know" state for dcache entries

The sequence of steps for a lookup with the possibility of "don't know" entries is conceptually straightforward:

  1. See if the object is already in the cache 
  2. If not: 
     1. allocate a new object, flagged as "locked"
     2. repeat the lookup, but this time insert the new object if none was found 
  3. If an existing object was found, free the new version (if we allocated it), then wait if the found object is locked 
  4. If no existing object was found, initialize the new object completely and unlock it, waking up any process waiting for it 

All of these steps can be seen in the new code, particularly in [`d_alloc_parallel()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/dcache.c?id=33688abb2802ff3a230bd2441f765477b94cc89e#n2414>), which covers 2a, 2b, and 3\. Step 4 can be found in [`lookup_slow()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=33688abb2802ff3a230bd2441f765477b94cc89e#n1608>). Step 1 is separate; it is part of the "fast path" executed when everything is in cache. It is embodied in various calls to `lookup_fast()`, such as the one in [`walk_component()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=33688abb2802ff3a230bd2441f765477b94cc89e#n1735>). The main source of extra complexity in this code is that a new hash table has been introduced to hold the "in-lookup" dentries. The primary hash table, `dentry_hashtable`, only holds entries on which lookup has completed and are thus known to be positive or negative; entries are added to the new `in_lookup_hash` using a separate linkage field (`d_u.d_in_lookup_hash`) in the dentry so that it can be transiently in both tables. When filesystem lookup completes, the entry is added to the primary hash table and then removed from the in-lookup hash table.

The lookup in step 2b needs to look in the primary hash table and then the in-lookup hash table, and it needs to be careful of possible races with the entry being moved from the latter to the former once lookup completes. To enable detection of these races [a new "bit-seq-lock" is introduced](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=84e710da2a1dfacfc87f604869a4d22df91ce6cd>) — like a [seqlock](<https://lwn.net/Articles/22818/>) but with a single bit used as the spinlock.

The value of the secondary hash table is that it allows the insertion of new entries without the need to search the hash chain in the primary table under an exclusive lock. An exclusive lock (obtained with `hlist_bl_lock()`) is needed to search the hash chain in the secondary table, but that can be expected to be a much shorter chain that is accessed much less often. The exclusive lock on the primary hash chain is only held long enough to attach the dentry once it is ready.

With these concerns in mind, step 2b above can be expanded to:

  1. Find the current value of the new per-directory bit-seq-lock 
  2. Search the primary hash table with only RCU protection — exit if found 
  3. Get an exclusive lock on the `in_lookup_hash` chain 
  4. Check whether the bit-seq-lock has changed. If it has, retry from A. If it hasn't, then we have not yet raced with the target dentry being moved between tables, and the lock we just took will stop the race from happening after this point 
  5. Search the `in_lookup_hash` chain; if nothing is found, insert the new entry that was allocated in 2a 

If the newly allocated dentry was inserted, a waitqueue provided by the caller is stored in the entry, in otherwise unused space, so a wakeup can be sent when the dentry is ready. If an existing, in-lookup dentry was found, then `d_alloc_parallel()` waits on that waitqueue for the wakeup, and then double checks to ensure that the dentry still looks correct: as no locks were held while waiting, the dentry could already have been renamed or unlinked.

With this understanding, it becomes possible to look through `d_alloc_parallel()` and most of it starts to make sense, though a particularly critical eye might notice

```
if (d_unhashed(dentry))
            continue;
```

in the middle of the loop performing the search in `in_lookup_hash`. A similar test appears in other loops that search in the primary hash table, so it is only surprising if you happen to remember that the two hash tables use different linkages and, as this function tests the linkage for the primary hash table, it really doesn't belong here.

This strangeness is particularly easy to notice with hindsight once you know that J. R. Okajima had been doing some testing [and reported problems with this code](<http://thread.gmane.org/13136.1466196630@jrobl>); together with Al Viro he had narrowed down the problem to exactly this line of code. Fortunately, it will now [be gone](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=e7d6ef9790bc281f5c29d0132b68031248523fe8>) before 4.7-final is released.

#### Replacing the exclusive lock with a shared lock

Once per-name locking is in place, [replacing the per-directory mutex](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=9902af79c01a8e39bb99b922fa3eef6d4ea23d69>) with a per-directory read/write semaphore and only taking a read (or shared) lock for lookup is fairly straightforward. It has had some interesting consequences though.

As previously reported, Jan Kara expressed some concern at LSFMM about the performance of semaphores. They are not widely used in the kernel and read/write semaphores are inherently more complex than mutexes, so performance regressions seemed at least a theoretical possibility. At the time, Viro reported that he hadn't been able to measure any, but more recently Dave Hansen has [found a small "blip" in unlink performance](<http://thread.gmane.org/5755D671.9070908@intel.com>) that he was able to narrow down to exactly the change from a mutex to a semaphore. Both mutexes and semaphores take an adaptive approach to waiting for the lock; first they spin for a little while, then they go to sleep and let the scheduler use the CPU for something else. They adapt slightly differently though, with mutexes spinning for longer in some cases. Consequently, using a mutex will waste more CPU time (reducing idle time) but often react more quickly (reducing latency).

Hansen wasn't really sure if this was an important regression or a puzzling inconsistency: ""Should we be making rwsems spin more, or mutexes spin less?"" he asked. Linus Torvalds [felt that the mutex was probably the right approach](<http://mid.gmane.org/CA+55aFxH_7wjo_BgUPK5iomWedE2=DaUZVX-yruHOWEk7OTiHQ@mail.gmail.com>) since performance matters and: ""Being slow under lock contention just tends to make for more lock contention"".

Meanwhile Waiman Long [has a patch set](<http://mid.gmane.org/5755E782.90800@hpe.com>) that makes a number of improvements to semaphores that may well address this issue too. So while the change was not as transparent as had been hoped, it appears that the performance of semaphores won't be a cause for concern for long.

In discussions following the original posting of this change, Viro [observed that](<http://mid.gmane.org/20160416032750.GW25498@ZenIV.linux.org.uk>):

FWIW, I agree that relying on i_mutex^Wi_rwsem for dcache protection is something worth getting rid of in the longer term. But that protection is there right now, and getting rid of that will take quite a bit of careful massage. 

So, if all goes well, the semaphore might eventually not be needed and any remaining measured regression will go along with it. 

The change from exclusive to shared locking brought up another performance issue of a different kind. This issue affects directory reads ("readdir") rather than lookup; readdir was changed to use shared locking at the same time that lookup was changed, and for many of the same reasons. In particular, it affects [`dcache_readdir()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/libfs.c?id=33688abb2802ff3a230bd2441f765477b94cc89e#n134>), which is used by filesystems that keep all entries in a directory in the dcache. Specifically, it affects tmpfs.

`dcache_readdir()` acquires the `d_lock` spinlock for the directory, and similar locks on the entries in the directory. Previously, when readdir held an exclusive lock on the directory's mutex, these locks would mostly be uncontended and so impose minimal cost. With only a shared lock it is possible for parallel readdir operations to experience much more contention on these locks. Usually, finer grained locking improves performance, but when those locks result in more contention events, it can work the other way. As Viro described it when he [reported the problem](<http://mid.gmane.org/20160606211522.GF14480@ZenIV.linux.org.uk>), there is now ""an obscene amount of grabbing/releasing `->d_lock` [...] And unlike mutex (or rswem exclusive), contention on ->d_lock chews a lot of cycles.""

This difficulty seems well on the way to being resolved with [a proposed patch](<http://mid.gmane.org/alpine.LFD.2.20.1606061649070.12258@i7>) that reduces the number of times that `d_lock` is claimed. It would not be fair to say that the shared-locking changes created this problem, but it does highlight that, when you make changes to locking rules, strange and unexpected results can certainly appear. This is why ongoing performance testing that looks for regressions, especially in unusual workloads, is so important; it is encouraging to see that happening.

There is clearly a lot of testing happening though, as Viro [observed separately in the context of some NFS-related races](<http://mid.gmane.org/20160604005611.GA14480@ZenIV.linux.org.uk>), ""we really need a consolidated regression testsuite"". Full coverage for network filesystems is more challenging than local filesystems, in part because it really requires multiple machines. Ad-hoc testing by the community certainly does find bugs, as we have seen here, but it seems that though we have much more structured testing than we once did, we would still benefit from having more.
