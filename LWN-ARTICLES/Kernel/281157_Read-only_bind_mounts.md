---
title: "Read-only bind mounts"
url: https://lwn.net/Articles/281157/
date: "May 6, 2008"
category: "Bind mounts; Filesystems-Mounting"
author: "By Jonathan Corbet May 6, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 6, 2008

Bind mounts can be thought of as a sort of symbolic link at the filesystem level. Using `mount --bind`, it is possible to create a second mount point for an existing filesystem, making that filesystem visible at a different spot in the namespace. Bind mounts are thus useful for creating specific views of the filesystem namespace; one can, for example, create a bind mount which makes a piece of a filesystem visible within an environment which is otherwise closed off with `chroot()`. 

There is one constraint to be found with bind mounts as implemented in kernels through 2.6.25, though: they have the same mount options as the primary mount. So a command like: 

```
mount --bind -o ro /vital_data /untrusted_container/vital_data
```

will fail to make `/vital_data` read-only under `/untrusted_container` if it was mounted writable initially. On your editor's 2.6.25 system, the failure is silent - the bind mount will be made writable despite the read-only request and no error message will be generated (the `mount` man page does document that options cannot be changed). 

There is clear value in the ability to make bind mounts read-only, though. Containers are one example: an administrator may wish to create a container in which processes may be running as root. It may be useful for that container to have access to filesystems on the host, but the container should not necessarily have write access to those filesystems. As of 2.6.26, this sort of configuration will be possible, thanks to the merging of the read-only bind mounts patches by Dave Hansen. 

As it happens, it's _still_ not possible to create a read-only bind mount with the command shown above; the read-only attribute can only be added with a remount operation afterward. So the necessary sequence is something like: 

```
mount --bind /vital_data /untrusted_container/vital_data
        mount -o remount,ro /untrusted_container/vital_data
```

This example raises an interesting question: what if some process opens a file for write access between the two `mount` operations? A system administrator has the right to expect that a read-only mount will, in fact, only be used for read operations. The 2.6.26 patch is designed to live up to that expectation, though the amount of work required turned out to be more than the developers might have expected. 

Filesystems normally track which files are opened for write access, so an attempt to remount a filesystem read-only can be passed to the low-level filesystem code for approval. But the low-level filesystem knows nothing about bind mounts, which are implemented entirely within the virtual filesystem (VFS) layer. So making read-only access for bind mounts work requires that the VFS keep track of all files which have been opened for write access. Or, more precisely, the VFS really only needs to keep track of _how many_ files are open for write access. 

The technique chosen was to create something which looks like a write lock for filesystems. Whenever the VFS is about to do something which involves writing, it must first call: 

```
int mnt_want_write(struct vfsmount *mnt);
```

The return value is zero if write access is possible, or a negative error code otherwise. This call can be found in obvious places - such as in the implementation of `open()` \- when write access is requested. But write access comes into play many other situations as well; for example, renaming a file requires write access for the duration of the operation. So `mnt_want_write()` calls have been sprinkled throughout the VFS code. 

When write access is no longer needed, the "write lock" should be released with a call to: 

```
void mnt_drop_write(struct vfsmount *mnt);
```

One of the discoveries which has been made is that write access is needed in rather more places than one might have thought. In particular, it turns out that there is need for `mnt_want_write()` calls within the low-level filesystems as well as in the VFS layer. So getting the read-only bind mounts patch into shape has been an ongoing process of finding the spots which have been missed and adding `mnt_want_write()` calls there. In an attempt to make this process a bit less error-prone, Miklos Szeredi has put together [a set of VFS helper functions](<http://lwn.net/Articles/280992/>) which encapsulate the situations where write access is needed. Those functions have not been merged for 2.6.26, however. 

Superficially, `mnt_want_write()` is easy to understand - it simply increments a counter of outstanding write accesses. The problem with a simple implementation, though, is that a shared, per-filesystem counter would create scalability problems. On multiprocessor systems, the cache line containing the counter would bounce around the system, slowing things considerably. 

A common response to this type of problem is to turn the counter into a per-CPU variable, allowing operations on the counter to remain local to each processor. When somebody needs to know the total value of the counters, it's a simple matter of adding each CPU's version; this operation is slow, but it is also rare. On big systems, though, the number of CPUs can be large - as can the number of filesystems, and bind mounts will only increase that number. The result is a multiplicative effect which, once again, is a scalability problem, only this time it manifests itself in the form of excessive memory use. 

The read-only bind mounts patch resolves this situation by, in effect, going back to global counters which are cached on specific processors. To that end, each CPU has one of these structures: 

```
struct mnt_writer {
    	spinlock_t lock;
    	unsigned long count;
    	struct vfsmount *mnt;
        }
```

At any given time, this structure will hold a local count for one filesystem, represented by `mnt`. If the processor needs to adjust the write count for that filesystem, it's a simple matter of incrementing or decrementing `count`. When the processor's attention turns to a different filesystem, it must first adjust the global count for the old filesystem, then it can switch its local `mnt_writer` structure to the new one. The result is a compromise between purely local and purely global counters which yields "good enough" performance on benchmarks designed to stress the system. 

Read-only bind mounts join with other features (such as [shared subtrees](<http://lwn.net/Articles/159077/>)) to create a flexible set of tools for the construction of the filesystem namespace. It is not clear how much of this functionality is being used at this time, but, as the implementation of containers in the mainline gets closer to completion, there is likely to be more interest in this capability. Linux systems in coming years may have much more complex filesystem layouts than have been seen in the past.
