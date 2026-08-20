---
title: "RCU-walk: faster pathname lookup in Linux"
url: https://lwn.net/Articles/649729/
date: "July 1, 2015"
category: "Filesystems-Virtual filesystem layer"
author: "July 1, 2015 This article was contributed by Neil Brown"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

July 1, 2015

This article was contributed by Neil Brown

[Previously](<https://lwn.net/Articles/649115/>) in this series we explored the pathname-lookup procedures in Linux (and the REF-walk mechanism in particular), which are complex because there are numerous details and special cases. This time we are looking at a different part of the process which is complex for a different reason. The read-copy-update-based RCU-walk mechanism avoids some details and cases by simply refusing to handle them; instead, it just falls back to REF-walk when it runs into something it cannot deal with. It remains hard to understand, though, because it is so unfamiliar. In an [earlier article](<https://lwn.net/Articles/453685/>) discussing this unfamiliarity I suggested that:

In human relationships a friendship can blossom more quickly if a mutual friend acts to introduce two parties and start them out on a sound footing. 

A couple of months ago, Al Viro — the maintainer of the Linux VFS layer — provided that introduction when he [took the time](<https://lkml.org/lkml/2015/4/23/432>), during a discussion of some possible changes, to put together a brief overview of the goals and mechanisms of RCU-walk. He also suggested that it ""probably needs to be turned into coherent text"" and be placed in `Documentation/filesystems/`. Not being one to turn down the opportunity to translate brief notes and C code into English, I took on the challenge. The first part of this series provided the context against which RCU-walk can make sense. The next part will detail the changes to symlink handling that were the concrete outcome of that discussion. This is the part where we make friends with RCU-walk.

RCU-walk is an algorithm for performing pathname lookup in Linux. It is in many ways similar to REF-walk, which we met last time, and the two share quite a bit of code. The significant difference in RCU-walk is how it allows for the possibility of concurrent access.

#### Clear demarcation of roles

The easiest way to manage concurrency is to forcibly stop any other thread from changing the data structures that a given thread is looking at. In cases where no other thread would even think of changing the data and lots of different threads want to read at the same time, this can be very costly. Even when using locks that permit multiple concurrent readers, the simple act of updating the count of the number of current readers can impose an unwanted cost. So the goal when reading a shared data structure that no other process is changing is to avoid writing anything to memory at all. Take no locks, increment no counts, leave no footprints.

The REF-walk mechanism already described certainly doesn't follow this principle, but then it is really designed to work when there may well be other threads modifying the data. RCU-walk, in contrast, is designed for the common situation where there are lots of frequent readers and only occasional writers. This may not be common in all parts of the filesystem tree, but in many parts it will be. For the other parts it is important that RCU-walk can quickly fall back to using REF-walk.

Pathname lookup always starts in RCU-walk mode but only remains there as long as what it is looking for is in the cache and is stable. It dances lightly down the cached filesystem image, leaving no footprints and carefully watching where it is, to be sure it doesn't trip. If it notices that something has changed or is changing, or if something isn't in the cache, then it tries to stop gracefully and switch to REF-walk.

This stopping requires getting a counted reference on the current `vfsmount` and `dentry`, and ensuring that these are still valid — that a path walk with REF-walk would have found the same entries. This is an invariant that RCU-walk must guarantee. It can only make decisions, such as selecting the next step, that are decisions which REF-walk could also have made if it were walking down the tree at the same time. If the graceful stop succeeds, the rest of the path is processed with the reliable, if slightly sluggish, REF-walk. If RCU-walk finds it cannot stop gracefully, it simply gives up and restarts from the top with REF-walk. 

This pattern of "try RCU-walk, if that fails try REF-walk" can be clearly seen in functions like [`filename_lookup()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2119>), [`filename_parentat()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2164>), [`filename_mountpoint()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2438>), [`do_filp_open()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n3321>), and [`do_file_open_root()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n3338>). These five correspond roughly to the four `path_`* functions we met last time, each of which calls [`link_path_walk()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1859>). The `path_`* functions are called using different mode flags until a mode is found which works. They are first called with `LOOKUP_RCU` set to request "RCU-walk". If that fails with the error `ECHILD` they are called again with no special flag to request "REF-walk". If either of those report the error `ESTALE` a final attempt is made with `LOOKUP_REVAL` set (and no `LOOKUP_RCU`) to ensure that entries found in the cache are forcibly revalidated -- normally entries are only revalidated if the filesystem determines that they are too old to trust.

The `LOOKUP_RCU` attempt may drop that flag internally and switch to REF-walk, but will never then try to switch back to RCU-walk. Places that trip up RCU-walk are much more likely to be near the leaves and so it is very unlikely that there will be much, if any, benefit from switching back.

#### RCU and seqlocks: fast and light

RCU is, unsurprisingly, critical to RCU-walk mode. The `rcu_read_lock()` is held for the entire time that RCU-walk is walking down a path. The particular guarantee it provides is that the key data structures — dentries, inodes, super_blocks, and mounts — will not be freed while the lock is held. They might be unlinked or invalidated in one way or another, but the memory will not be repurposed, so values in various fields will still be meaningful. This is the only guarantee that RCU provides; everything else is done using seqlocks.

As we saw last time, REF-walk holds a counted reference to the current `dentry` and the current `vfsmount`, and does not release those references before taking references to the "next" dentry or vfsmount. It also sometimes takes the `d_lock` spinlock. These references and locks are taken to prevent certain changes from happening. RCU-walk must not take those references or locks and so cannot prevent such changes. Instead, it checks to see if a change has been made, and aborts or retries if it has.

To preserve the invariant mentioned above (that RCU-walk may only make decisions that REF-walk could have made), it must make the checks at or near the same places that REF-walk holds the references. So, when REF-walk increments a reference count or takes a spinlock, RCU-walk samples the status of a seqlock using `read_seqcount_begin()` or a similar function. When REF-walk decrements the count or drops the lock, RCU-walk checks if the sampled status is still valid using `read_seqcount_retry()` or similar. 

However, there is a little bit more to seqlocks than that. If RCU-walk accesses two different fields in a seqlock-protected structure, or accesses the same field twice, there is no a-priori guarantee of any consistency between those accesses. When consistency is needed — which it usually is — RCU-walk must take a copy and then use `read_seqcount_retry()` to validate that copy.

`read_seqcount_retry()` not only checks the sequence number, but also imposes a [memory barrier](<https://www.kernel.org/doc/Documentation/memory-barriers.txt>) so that no memory-read instruction from _before_ the call can be delayed until _after_ the call, either by the CPU or by the compiler. A simple example of this can be seen in [`slow_dentry_cmp()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/dcache.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2079>) which, for filesystems which do not use simple byte-wise name equality, calls into the filesystem to compare a name against a dentry. The length and name pointer are copied into local variables, then `read_seqcount_retry()` is called to confirm the two are consistent, and only then is `->d_compare()` called. When standard filename comparison is used, [`dentry_cmp()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/dcache.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n215>) is called instead. Notably it does _not_ use `read_seqcount_retry()`, but instead has a large comment explaining why the consistency guarantee isn't necessary. A subsequent `read_seqcount_retry()` will be sufficient to catch any problem that could occur at this point.

With that little refresher on seqlocks out of the way we can look at the bigger picture of how RCU-walk uses seqlocks.

#### `mount_lock` and `nd->m_seq`

We already met the `mount_lock` seqlock when REF-walk used it to ensure that crossing a mount point is performed safely. RCU-walk uses it for that too, but for quite a bit more.

Instead of taking a counted reference to each `vfsmount` as it descends the tree, RCU-walk samples the state of `mount_lock` at the start of the walk and stores this initial sequence number in the `struct nameidata` in the `m_seq` field. This one lock and one sequence number are used to validate all accesses to all `vfsmounts` and all mount point crossings. As changes to the mount table are relatively rare, it is reasonable to fall back on REF-walk any time that any "mount" or "unmount" happens.

`m_seq` is checked (using `read_seqretry()`) at the end of an RCU-walk sequence, whether switching to REF-walk for the rest of the path or when the end of the path is reached. It is also checked when stepping down over a mount point (in [`__follow_mount_rcu()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1235>)) or up (in [`follow_dotdot_rcu()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1279>)). If it is ever found to have changed, the whole RCU-walk sequence is aborted and the path is processed again by REF-walk.

If RCU-walk finds that `mount_lock` hasn't changed then it can be sure that, had REF-walk taken counted references on each `vfsmount`, the results would have been the same. This ensures the invariant holds, at least for `vfsmount` structures.

#### `dentry->d_seq` and `nd->seq`

In place of taking a count or lock on `d_reflock`, RCU-walk samples the per-`dentry` `d_seq` seqlock, and stores the sequence number in the `seq` field of the `nameidata` structure, so `nd->seq` should always be the current sequence number of `nd->dentry`. This number needs to be revalidated after copying, and before using, the name, parent, or inode of the `dentry`.

The handling of the name we have already looked at, and the parent is only accessed in `follow_dotdot_rcu()` which fairly trivially follows the required pattern, though it does so for three different cases.

When not at a mount point, `d_parent` is followed and its `d_seq` is collected. When we are at a mount point, we instead follow the `mnt->mnt_mountpoint` link to get a new `dentry` and collect its `d_seq`. Then, after finally finding a `d_parent` to follow, we must check if we have landed on a mount point and, if so, must find that mount point and follow the `mnt->mnt_root` link. This would imply a somewhat unusual, but certainly possible, circumstance where the starting point of the path lookup was in part of the filesystem that was mounted on, and so not visible from the root.

The inode pointer, stored in `->d_inode`, is a little more interesting. The inode will always need to be accessed at least twice, once to determine if it is NULL and once to verify access permissions. Symlink handling requires a validated inode pointer too. Rather than revalidating on each access, a copy is made on the first access and it is stored in the `inode` field of `nameidata` from where it can be safely accessed without further validation.

`lookup_fast()` is the only lookup routine that is used in RCU-mode, `lookup_slow()` being too slow and requiring locks. It is in [`lookup_fast()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1503>) that we find the important "hand over hand" tracking of the current `dentry`.

The current `dentry` and current `seq` number are passed to [`__d_lookup_rcu()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/dcache.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2117>) which, on success, returns a new `dentry` and a new `seq` number. `lookup_fast()` then copies the inode pointer and revalidates the new `seq` number. It then validates the old `dentry` with the old `seq` number one last time and only then continues. This process of getting the `seq` number of the new `dentry` and then checking the `seq` number of the old exactly mirrors the process of getting a counted reference to the new `dentry` before dropping that for the old `dentry` which we saw in REF-walk.

#### No `inode->i_mutex` or even `rename_lock`

A mutex is a fairly heavyweight lock that can only be taken when it is permissible to sleep. As `rcu_read_lock()` forbids sleeping, `inode->i_mutex` plays no role in RCU-walk. If some other thread does take `i_mutex` and modifies the directory in a way that RCU-walk needs to notice, the result will be either that RCU-walk fails to find the dentry that it is looking for, or it will find a dentry which `read_seqretry()` won't validate. In either case it will drop down to REF-walk mode which can take whatever locks are needed. 

Though `rename_lock` could be used by RCU-walk as it doesn't require any sleeping, RCU-walk doesn't bother. REF-walk uses `rename_lock` to protect against the possibility of hash chains in the dcache changing while they are being searched. This can result in failing to find something that actually is there. When RCU-walk fails to find something in the dentry cache, whether it is really there or not, it already drops down to REF-walk and tries again with appropriate locking. This neatly handles all cases, so adding extra checks on `rename_lock` would bring no significant value. 

#### `unlazy walk()` and `complete_walk()`

That "dropping down to REF-walk" typically involves a call to `unlazy_walk()`, so named because "RCU-walk" is also sometimes referred to as "lazy walk". [`unlazy_walk()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n648>) is called when following the path down to the current `vfsmount`/`dentry` pair seems to have proceeded successfully, but the next step is problematic. This can happen if the next name cannot be found in the dcache, if permission checking or name revalidation couldn't be achieved while the `rcu_read_lock()` is held (which forbids sleeping), if an automount point is found, or in a couple of cases involving symlinks. It is also called from [`complete_walk()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n751>) when the lookup has reached the final component, or the very end of the path, depending on which particular flavor of lookup is used.

Other reasons for dropping out of RCU-walk that do not trigger a call to `unlazy_walk()` are when some inconsistency is found that cannot be handled immediately, such as `mount_lock` or one of the `d_seq` seqlocks reporting a change. In these cases the relevant function will return `-ECHILD` which will percolate up until it triggers a new attempt from the top using REF-walk.

For those cases where `unlazy_walk()` is an option, it essentially takes a reference on each of the pointers that it holds (`vfsmount`, dentry, and possibly some symbolic links) and then verifies that the relevant seqlocks have not been changed. If there have been changes, it, too, aborts with `-ECHILD`, otherwise the transition to REF-walk has been a success and the lookup process continues. 

Taking a reference on those pointers is not quite as simple as just incrementing a counter. That works to take a second reference if you already have one (often indirectly through another object), but it isn't sufficient if you don't actually have a counted reference at all. For `dentry->d_lockref`, it is safe to increment the reference counter to get a reference unless it has been explicitly marked as "dead" which involves setting the counter to `-128`. [`lockref_get_not_dead()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/lib/lockref.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n169>) achieves this.

For `mnt->mnt_count` it is safe to take a reference as long as `mount_lock` is then used to validate the reference. If that validation fails, it may _not_ be safe to just drop that reference in the standard way of calling `mnt_put()` — an unmount may have progressed too far. So the code in [`legitimize_mnt()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namespace.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n591>), when it finds that the reference it got might not be safe, checks the `MNT_SYNC_UMOUNT` flag to determine if a simple `mnt_put()` is correct, or if it should just decrement the count and pretend none of this ever happened.

#### Taking care in filesystems

RCU-walk depends almost entirely on cached information and often will not call into the filesystem at all. However there are two places, besides the already-mentioned component-name comparison, where the file system might be included in RCU-walk, and it must know to be careful.

If the filesystem has non-standard permission-checking requirements — such as a networked filesystem which may need to check with the server — the `i_op->permission` interface might be called during RCU-walk. In this case an extra "`MAY_NOT_BLOCK`" flag is passed so that it knows not to sleep, but to return `-ECHILD` if it cannot complete promptly. `i_op->permission` is given the inode pointer, not the dentry, so it doesn't need to worry about further consistency checks. However if it accesses any other filesystem data structures, it must ensure they are safe to be accessed with only the `rcu_read_lock()` held. This typically means they must be freed using `kfree_rcu()` or similar.

If the filesystem may need to revalidate dcache entries, then `d_op->d_revalidate` may be called in RCU-walk too. This interface _is_ passed the dentry but does not have access to the `inode` or the `seq` number from the `nameidata`, so it needs to be extra careful when accessing fields in the `dentry`. This extra care typically involves using `ACCESS_ONCE()` or the newer [`READ_ONCE()`](<https://lwn.net/Articles/624126/>) to access fields, and verifying the result is not NULL before using it. This pattern can be see in [`nfs_lookup_revalidate()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/nfs/dir.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1100>).

#### A pair of patterns

In various places in the details of REF-walk and RCU-walk, and also in the big picture, there are a couple of related patterns that are worth being aware of.

The first is "try quickly and check, if that fails, try slowly". We can see that in the high-level approach of first trying RCU-walk and then trying REF-walk, and in places were `unlazy_walk()` is used to switch to REF-walk for the rest of the path. We also saw it last time in `dget_parent()` when following a "`..`" link. It tries a quick way to get a reference, then falls back to taking locks if needed.

The second pattern is "try quickly and check, if that fails, try again — repeatedly". This is seen with the use of `rename_lock` and `mount_lock` in REF-walk. RCU-walk doesn't make use of this pattern; if anything goes wrong it is much safer to just abort and try a more sedate approach.

The emphasis here is "try quickly and check". It should probably be "try quickly _and carefully_ , then check". The fact that checking is needed is a reminder that the system is dynamic and only a limited number of things are safe at all. The most likely cause of errors in this whole process is assuming something is safe when in reality it isn't. Careful consideration of what exactly guarantees the safety of each access is sometimes necessary.

#### Next: symlinks

We have now covered nearly all of pathname lookup with the major missing part being symbolic links. The handling of symbolic links received a major rewrite recently so it deserves a thorough treatment. That treatment will form the bulk of the final article in this series which should appear in the next week or so.
