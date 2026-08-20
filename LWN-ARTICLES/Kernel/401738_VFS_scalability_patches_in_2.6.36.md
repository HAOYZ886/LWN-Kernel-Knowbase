---
title: VFS scalability patches in 2.6.36
url: https://lwn.net/Articles/401738/
date: "August 24, 2010"
category: "Filesystems-Virtual filesystem layer; Scalability"
author: "By Jonathan Corbet August 24, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 24, 2010

It is rare for Linus to talk about what he plans to merge in a given development cycle before the merge window opens; it seems that he prefers to see what the pull requests look like and make his decisions afterward. He made an exception in [the 2.6.35 announcement](<http://lwn.net/Articles/398371/>), though: 

On a slightly happier note: one thing I do hope we can merge in the upcoming merge window is Nick Piggin's cool VFS scalability series. I've been using it on my own machine, and gone through all the commits (not that I shouldn't go through some of them some more), and am personally really excited about it. It's seldom we see major performance improvements in core code that are quite that noticeable, and Nick's whole RCU pathname lookup in particular just tickles me pink. 

It's a rare developer who, upon having tickled the Big Penguin to that particular shade, will hold off on merging his changes. But Nick [asked that the patches sit out for one more cycle](<https://lwn.net/Articles/401743/>), perhaps out of the entirely rational fear of bugs which might irritate users to a rather deeper shade. So Linus will have to wait a bit for his RCU pathname lookup code. That said, some parts of the VFS scalability code did make it into the mainline for 2.6.36-rc2. 

Like most latter-day scalability work, the VFS work is focused on increasing locality and eliminating situations where CPUs must share resources. Given that a filesystem is an inherently global structure, increasing locality can be a challenging task; as a result, parts of Nick's patch set are on the complex and tricky side. But, in the end, it comes down to dealing with things locally whenever possible, but making global action possible when the need arises. 

The first step is the introduction of two new lock types, the first of which is called a "local/global lock" (lglock). An lglock is intended to provide very fast access to per-CPU data while making it possible (at a rather higher cost) to get at another CPU's data. An lglock is created with: 

```
#include <linux/lglock.h>
    
        DEFINE_LGLOCK(name);
```

The `DEFINE_LGLOCK()` macro is a 99-line wonder which creates the necessary data structure and accessor functions. By design, lglocks can only be defined at the file global level; they are not intended to be embedded within data structures. 

Another set of macros is used for working with the lock: 

```
lg_lock_init(name);
        lg_local_lock(name);
        lg_local_unlock(name);
        lg_local_lock_cpu(name, int cpu);
        lg_local_unlock_cpu(name, int cpu);
```

Underneath it all, an lglock is really just a per-CPU array of spinlocks. So a call to `lg_local_lock()` will acquire the current CPU's spinlock, while `lg_local_lock_cpu()` will acquire the lock belonging to the specified `cpu`. Acquiring an lglock also disables preemption, which would not otherwise happen in realtime kernels. As long as almost all locking is local, it will be very fast; the lock will not bounce between CPUs and will not be contended. Both of those assumptions go away, of course, if the cross-CPU version is used. 

Sometimes it is necessary to globally lock the lglock: 

```
lg_global_lock(name);
        lg_global_unlock(name);
        lg_global_lock_online(name);
        lg_global_unlock_online(name);
```

A call to `lg_global_lock()` will go through the entire array, acquiring the spinlock for every CPU. Needless to say, this will be a very expensive operation; if it happens with any frequency at all, an lglock is probably the wrong primitive to use. The `_online` version only acquires locks for CPUs which are currently running, while `lg_global_lock()` acquires locks for all possible CPUs. 

The VFS scalability patch set also brings back the "big reader lock" concept. The idea behind a brlock is to make locking for read access as fast as possible, while making write locking possible. The brlock API (also defined in `<linux/lglock.h>`) looks like this: 

```
DEFINE_BRLOCK(name);
    
        br_lock_init(name);
        br_read_lock(name);
        br_read_unlock(name);
        br_write_lock(name);
        br_write_unlock(name);
```

As it happens, this version of brlocks is implemented entirely with lglocks; `br_read_lock()` maps directly to `lg_local_lock()`, and `br_write_lock()` turns into `lg_global_lock()`. 

The first use of lglocks is to protect the list of open files which is attached to each superblock structure. This list is currently protected by the global `files_lock`, which becomes a bottleneck when a lot of `open()` and `close()` calls are being made. In 2.6.36, the list of open files becomes a per-CPU array, with each CPU managing its own list. When a file is opened, a (cheap) call to `lg_local_lock()` suffices to protect the local list while the new file is added. 

When a file is closed, things are just a bit more complicated. There is no guarantee that the file will be on the local CPU's list, so the VFS must be prepared to reach across to another CPU's list to clean things up. That, of course, is what `lg_local_lock_cpu()` is for. Cross-CPU locking will be more expensive than local locking, but (1) it only involves one other CPU, and (2) in situations where there is a lot of opening and closing of files, chances are that the process working with any specific file will not migrate between CPUs during the (presumably short) time that the file is open. 

The real reason that the per-superblock open files list exists is to let the kernel check for writable files when a filesystem is being remounted read-only. That operation requires exclusive access to the entire list, so `lg_global_lock()` is used. The global lock is costly, but read-only remounts are not a common occurrence, so nobody is likely to notice. 

Also for 2.6.36, Nick changed the global `vfsmount_lock` into a brlock. This lock protects the tree of mounted filesystems; it must be acquired (in a read-only mode) whenever a pathname lookup crosses from one mount point to the next. Write access is only needed when filesystems are mounted or unmounted - again, an uncommon occurrence on most systems. Nick warns that this change is unlikely to speed up most workloads now - indeed, it may slow some down slightly - but its value will become clearer when some of the other bottlenecks are taken care of. 

Aside from a few smaller changes, that is where VFS scalability work stops for the 2.6.36 development cycle. The more complicated work - dealing with `dcache_lock` in particular - will go through a few more months of testing before it is pushed toward the mainline. Then, perhaps, we'll see Linus in a proper shade of pink.
