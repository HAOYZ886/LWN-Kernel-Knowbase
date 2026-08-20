---
title: Unionfs
url: https://lwn.net/Articles/217084/
date: "January 10, 2007"
category: "Filesystems-Union; Unionfs"
author: 
---

A longstanding (and long unsupported in Linux) filesystem concept is that of a union filesystem. In brief, a union filesystem is a logical combination of two or more other filesystems to create the illusion of a single filesystem with the contents of all the others. 

As an example, imagine that a user wanted to mount a distribution DVD full of packages. It would be nice to be able to add updated packages to close today's security holes, but the DVD is a read-only medium. The solution is a union filesystem. A system administrator can take a writable filesystem and join it with the read-only DVD, creating a writable filesystem with the contents of both. If the user then adds packages, they will go into the writable filesystem, which can be smaller than would be needed if it were to hold the entire contents. 

The [unionfs patch](<http://lwn.net/Articles/216761/>) posted by Josef Sipek provides this capability. With unionfs in place, the system administrator could construct the union with a command sequence like: 

```
mount -r /dev/dvd /mnt/media/dvd
        mount    /dev/hdb1 /mnt/media/dvd-overlay
        mount -t unionfs \
              -o dirs=/mnt/media/dvd-overlay=rw:/mnt/media/dvd=ro \
              /writable-dvd
```

The first two lines just mount the DVD and the writable partition as normal filesystems. The final command then joins them into a single union, mounted on `/writable-dvd`. Each "branch" of a union has a priority, determined by the order in which they are given in the `dirs=` option. When a file is looked up, the branches are searched in priority order, with the first occurrence found being returned to the user. If an attempt is made to write a read-only file, that file will be copied into the highest-priority writable branch and written there. 

As one might imagine, there is a fair amount of complexity required to make all of this actually work. Joining together filesystem hierarchies, copying files between them, and inserting "whiteouts" to mask files deleted from read-only branches are just a few of the challenges which must be met. The unionfs code seems to handle most of them well, providing convincing Unix semantics in the joined filesystem. 

Reviewers immediately jumped on one exception, which was noted in the documentation: 

Modifying a Unionfs branch directly, while the union is mounted, is currently unsupported. Any such change can cause Unionfs to oops, or stay silent and even RESULT IN DATA LOSS. 

What this means is that it is dangerous to mess directly with the filesystems which have been joined into a union mount. Andrew Morton pointed out that, as user-friendly interfaces go, this one is a little on the rough side. Since bind mounts don't have this problem, he asked, why should unionfs present such a trap to its users? Josef [responded](<https://lwn.net/Articles/217090/>): 

Bind mounts are a purely VFS level construct. Unionfs is, as the name implies, a filesystem. Last year at OLS, it seemed that a lot of people agreed that unioning is neither purely a fs construct, nor purely a vfs construct. 

That, in turn, led to some fairly definitive statements that unionfs _should_ be implemented at the virtual filesystem level. Without that, it's not clear that it will ever be possible to keep the namespace coherent in the face of modifications at all levels of the union. So it seems clear that, to truly gain the approval of the kernel developers, unionfs needs a rewrite. Andrew Morton has been [heard to wonder](<https://lwn.net/Articles/217092/>) if the current version should be merged anyway in the hopes that it would help inspire that rewrite to happen. No decisions have been made as of this writing, so it's far from clear whether Linux will have unionfs support in the near future or not.
