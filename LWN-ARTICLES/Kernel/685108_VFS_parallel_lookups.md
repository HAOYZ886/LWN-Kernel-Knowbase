---
title: VFS parallel lookups
url: https://lwn.net/Articles/685108/
date: "April 27, 2016"
category: "Filesystems-Virtual filesystem layer"
author: "By Jake Edge April 27, 2016 LSFMM 2016"
---

By **Jake Edge**  
April 27, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

In one of just a handful of filesystem-only sessions at the 2016 Linux Storage, Filesystem, and Memory-Management Summit, Al Viro reported on work he has done to allow VFS lookups to proceed in parallel. Today, all directory operations are done with the inode mutex (`i_mutex`) held, which prevents anything else from touching that directory. But the most common operation, lookup, is non-destructive, so there is no real conceptual reason to stop it from happening in parallel. Solving that scalability problem took some work, though, as Viro described. 

[ ![\[Al Viro\]](https://static.lwn.net/images/2016/lsf-viro-sm.jpg) ](<https://lwn.net/Articles/685159/>)

The obvious choice to replace the mutex is a read/write semaphore (rwsem), Viro said. But there is a problem: the mutex currently protects the directory entry (dentry). A lookup operation can cause dentries to be created, which can lead to races if two dentries are created for the same name. Unwinding that took some effort, he said. There is a need to ensure that there are never two hashed dentries with the same parent and name at the same time. If that were to happen, subsequent lookups would only find one or the other, which must be avoided. If two lookups on the same name in the same directory run in parallel, there is a danger that these two dentries would be created. 

There was a need for an object that would be used to indicate that a lookup was in progress for a given parent and name. When a dentry is not found in the dentry cache, a new dentry is created to be passed to the filesystem `lookup()` function. That dentry is the obvious place to track a lookup in progress for the given parent and name. There are some fields that are unused at that point, so they can be repurposed for lookup tracking. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

These in-progress lookup dentries are tracked in a hash on the parent. That hash can't be bigger than the number of in-progress lookups for that directory. If a lookup finds an entry on the parent's hash for the same name, it simply waits until the earlier lookup is done. So there are no parallel lookups for the same parent/name combination. 

His "lookups" branch that implements parallel lookups "actually works", Viro said. `i_mutex` is replaced with `i_rwsem` and lookups are done using that shared lock. 

For `readdir()`, a different choice was made. Because there is state associated with `readdir()` (i.e. directory position), it doesn't really make sense to allow two threads to be calling `getdents()` on the same directory file descriptor in parallel. The `struct file` that represents the open directory file has a lock that prevents `read()` and `lseek()` from happening in parallel; it is used to prevent parallel `readdir()`/`getdents()` calls. 

One problem that he ran into when converting to an rwsem is a lack of ["killable" variants](<https://lwn.net/Articles/288056/>) of the semaphore primitives (like `down_write()`). There is a patch series floating around that adds `down_write_killable()`, but it has not stabilized yet so, for now, he replaced `mutex_lock_killable()` calls with `down_write()`, which is fine for testing purposes. 

Jan Kara asked about the performance of the semaphore. Viro said that there have been no performance regressions that he has seen on the tests he runs. But read/write semaphores are a bit costlier. Kara was concerned that all lookups are paying the cost of the semaphore, when only some lookups get the benefit of parallelism. Hugh Dickins said that a lot of effort has been put into improving the performance of the semaphore, so the differences should be minimal.
