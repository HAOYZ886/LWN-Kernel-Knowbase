---
title: Pathname lookup in Linux
url: https://lwn.net/Articles/649115/
date: "June 24, 2015"
category: "Filesystems-Virtual filesystem layer"
author: "June 24, 2015 This article was contributed by Neil Brown"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

June 24, 2015

This article was contributed by Neil Brown

One of the changes that was [recently merged](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801>) for Linux 4.2 is a substantial rewrite of parts of the pathname-lookup code in the Linux virtual filesystem (VFS) layer. This rewrite primarily affects the handling of symbolic links -- though, like many such rewrites, opportunities were found to rationalize and improve other parts of the code, too. With all this change, it seems like a good opportunity to document just how pathname lookup works. While such documentation cannot stay current indefinitely, writing it immediately after a big change might increase the time until it becomes inaccurate.

The most obvious aspect of pathname lookup, which very little exploration is needed to discover, is that it is complex. There are many rules, special cases, and implementation alternatives that all combine to confuse the unwary reader. Computer science has long been acquainted with such complexity and has tools to help manage it. One tool that we will make extensive use of is "divide and conquer". For the early parts of the analysis we will divide off symlinks, leaving them until the final part. Well before we get to symlinks we have another major division based on the VFS's approach to locking which will allow us to review "REF-walk" and "RCU-walk" (a pair of algorithms that have been [described previously](<https://lwn.net/Articles/419811/>)) separately. But we are getting ahead of ourselves. There are some important low-level distinctions we need to clarify first.

#### There are two sorts of ...

Pathnames (sometimes "file names"), used to identify objects in the filesystem, will be familiar to most readers. They contain two sorts of elements: "slashes" that are sequences of one or more "`/`" characters, and "components" that are sequences of one or more non-"`/`" characters. These form two kinds of paths. Those that start with slashes are "absolute" and start from the filesystem root. The others are "relative" and start from the current directory, or from some other location specified by a file descriptor given to a "xxx`at()`" system call such as [`openat()`](<http://man7.org/linux/man-pages/man2/openat.2.html>).

It is tempting to describe the second kind as starting with a component, but that isn't always accurate: a pathname can lack both slashes and components; it can be empty, in other words. This is generally forbidden in POSIX, but some of those "xxx`at()`" system calls in Linux permit it when the `AT_EMPTY_PATH` flag is given. For example, if you have an open file descriptor on an executable file, you can execute it by calling [`execveat()`](<http://man7.org/linux/man-pages/man2/execveat.2.html>), passing the file descriptor, an empty path, and the `AT_EMPTY_PATH` flag.

These paths can be divided into two sections: the final component and everything else. The "everything else" is the easy bit. In all cases it must identify a directory that already exists, otherwise an error such as `ENOENT` or `ENOTDIR` will be reported. The final component is not so simple. Not only do different system calls interpret it quite differently (e.g. some create it, some do not), but it might not even exist: neither the empty pathname nor the pathname that is just slashes have a final component. If it does exist, it could be "`.`" or "`..`" which are handled quite differently from other components.

If a pathname ends with a slash, such as "`/tmp/foo/`" it might be tempting to consider that to have an empty final component. In many ways that would lead to correct results, but not always. In particular, `mkdir()` and `rmdir()` each create or remove a directory named by the final component, and they are required to work with pathnames ending in "`/`". According to [POSIX](<http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap04.html#tag_04_12>): 

A pathname that contains at least one non-<slash> character and that ends with one or more trailing <slash> characters shall not be resolved successfully unless the last pathname component before the trailing <slash> characters names an existing directory or a directory entry that is to be created for a directory immediately after the pathname is resolved.

The Linux pathname-walking code (mostly in `fs/namei.c`) deals with all of these issues: breaking the path into components, handling the "everything else" quite separately from the final component, and checking that the trailing slash is not used where it isn't permitted. It also addresses the important issue of concurrent access.

While one process is looking up a pathname, another might be making changes that affect that lookup. One fairly extreme case is that if `a/b` were renamed to `a/c/b` while another process were looking up `a/b/..`, that process might successfully resolve on `a/c`. Most races are much more subtle, and a big part of the task of pathname lookup is to prevent them from having damaging effects. Many of the possible races are seen most clearly in the context of the "dcache" and an understanding of that is central to understanding pathname lookup.

#### More than just a cache

The dcache caches information about names in each filesystem to make them quickly available for lookup. Each entry (known as a "[dentry](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/include/linux/dcache.h?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n107>)") contains three significant fields: a component name, a pointer to a parent dentry, and a pointer to the "inode" which contains further information about the object in that parent with the given name. The inode pointer can be `NULL` indicating that the name doesn't exist in the parent. While there can be linkage in the dentry of a directory to the dentries of the children, that linkage is not used for pathname lookup, and so will not be considered here.

The dcache has a number of uses apart from accelerating lookup. One that will be particularly relevant is that it is closely integrated with the mount table that records which filesystem is mounted where. What the mount table actually stores is which dentry is mounted on top of which other dentry.

When considering the dcache, we have another of our "two types" distinctions: there are two types of filesystems. Some filesystems ensure that the information in the dcache is always completely accurate (though not necessarily complete). This can allow the VFS to determine if a particular file does or doesn't exist without checking with the filesystem, and means that the VFS can protect the filesystem against certain races and other problems. These are typically "local" filesystems such as ext3, XFS, and Btrfs.

Other filesystems don't provide that guarantee because they cannot. These are typically filesystems that are shared across a network, whether remote filesystems like NFS and 9P, or cluster filesystems like ocfs2 or cephfs. These filesystems allow the VFS to revalidate cached information and must provide their own protection against awkward races. The VFS can detect these filesystems by the `DCACHE_OP_REVALIDATE` flag being set in the dentry.

#### REF-walk: simple concurrency management with refcounts and spinlocks

With all of those divisions carefully classified, we can now start looking at the actual process of walking along a path. In particular we will start with the handling of the "everything else" part of a pathname, and focus on the "REF-walk" approach to concurrency management. This code is found in the [`link_path_walk()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1859>) function, if you ignore all the places that only run when "`LOOKUP_RCU`" (indicating the use of RCU-walk) is set.

REF-walk is fairly heavy-handed with locks and reference counts. Not as heavy-handed as in the old "big kernel lock" days, but certainly not afraid of taking a lock when one is needed. It uses a variety of different concurrency controls. A background understanding of the various primitives is assumed, or can be gleaned from elsewhere such as in [Meet the Lockers](<https://lwn.net/Articles/453685/>).

The locking mechanisms used by REF-walk include:

`dentry->d_lockref`

     This uses the relatively recently introduced [lockref](<https://lwn.net/Articles/565734/>) primitive to provide both a spinlock and a reference count. The special sauce of this primitive is that the conceptual sequence "lock; inc_ref; unlock;" can often be performed with a single, atomic memory operation.

Holding a reference on a dentry ensures that the dentry won't suddenly be freed and used for something else, so the values in various fields will behave as expected. It also protects the `->d_inode` reference to the inode to some extent.

The association between a dentry and its inode is fairly permanent. For example, when a file is renamed, the dentry and inode move together to the new location. When a file is created the dentry will initially be negative (i.e. `d_inode` is `NULL`), and will be assigned to the new inode as part of the act of creation.

When a file is deleted, this can be reflected in the cache either by setting `d_inode` to `NULL`, or by removing it from the hash table (described shortly) used to look up the name in the parent directory. If the dentry is still in use, the second option is used, as it is perfectly legal to keep using an open file after it has been deleted; having the dentry around helps. If the dentry is not otherwise in use (i.e. if the refcount in `d_lockref` is one), only then will `d_inode` be set to `NULL`. Doing it this way is more efficient for a very common case.

So as long as a counted reference is held to a dentry, a non-NULL `->d_inode` value will never be changed. 

`dentry->d_lock`

     `d_lock` is a synonym for the spinlock that is part of `d_lockref` above. For our purposes, holding this lock protects against the dentry being renamed or unlinked. In particular, its parent (`d_parent`), and its name (`d_name`) cannot be changed, and it cannot be removed from the dentry hash table.

When looking for a name in a directory, REF-walk takes `d_lock` on each candidate dentry that it finds in the hash table and then checks that the parent and name are correct. So it doesn't lock the parent while searching in the cache; it only locks children.

When looking for the parent for a given name (to handle "`..`"), REF-walk can take `d_lock` to get a stable reference to `d_parent`, but it first tries a more lightweight approach. As seen in [`dget_parent()`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/dcache.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n810>), if a reference can be claimed on the parent, and if subsequently `d_parent` can be seen to have not changed, then there is no need to actually take the lock on the child.

`rename_lock`

     Looking up a given name in a given directory involves computing a hash from the two values (the name and the dentry of the directory), accessing that slot in a hash table, and searching the linked list that is found there. When a dentry is renamed, the name and the parent dentry can both change so the hash will almost certainly change too. This would move the dentry to a different chain in the hash table. If a filename search happened to be looking at a dentry that was moved in this way, it might end up continuing the search down the wrong chain, and so miss out on part of the correct chain.

The name-lookup process (`[d_lookup()](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/dcache.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2223>)`) does _not_ try to prevent this from happening, but only to detect when it happens. `rename_lock` is a seqlock that is updated whenever any dentry is renamed. If `d_lookup` finds that a rename happened while it unsuccessfully scanned a chain in the hash table, it simply tries again.

`inode->i_mutex`

     `i_mutex` is a mutex that serializes all changes to a particular directory. This ensures that, for example, an `unlink()` and a `rename()` cannot both happen at the same time. It also keeps the directory stable while the filesystem is asked to look up a name that is not currently in the dcache.

This has a complementary role to that of `d_lock`: `i_mutex` on a directory protects all of the names in that directory, while `d_lock` on a name protects just one name in a directory. Most changes to the dcache hold `i_mutex` on the relevant directory inode and briefly take `d_lock` on one or more the dentries while the change happens. One exception is when idle dentries are removed from the dcache due to memory pressure. This uses `d_lock`, but `i_mutex` plays no role.

The mutex affects pathname lookup in two distinct ways. Firstly, it serializes lookup of a name in a directory. `[walk_component()](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1695>)` uses `[lookup_fast()](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1503>)` first which, in turn, checks to see if the name is in the cache, using only `d_lock` locking. If the name isn't found, then `walk_component()` falls back to `[lookup_slow()](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1601>)`, which takes `i_mutex`, checks again that the name isn't in the cache, and then calls in to the filesystem to get a definitive answer. A new dentry will be added to the cache regardless of the result.

Secondly, when pathname lookup reaches the final component, it will sometimes need to take `i_mutex` before performing the last lookup so that the required exclusion can be achieved. How path lookup chooses to take, or not take, `i_mutex` is one of the issues addressed in a subsequent section.

`mnt->mnt_count`

     `mnt_count` is a per-CPU reference counter on "`mount`" structures. Per-CPU here means that incrementing the count is cheap as it only uses CPU-local memory, but checking if the count is zero is expensive as it needs to check with every CPU. Taking a `mnt_count` reference prevents the `mount` structure from disappearing as the result of regular unmount operations, but does not prevent a "lazy" unmount. So holding `mnt_count` doesn't ensure that the mount remains in the namespace and, in particular, doesn't stabilize the link to the mounted-on dentry. It does, however, ensure that the `mount` data structure remains coherent, and it provides a reference to the root dentry of the mounted filesystem. So a reference through `->mnt_count` provides a stable reference to the mounted dentry, but not the mounted-on dentry.

`mount_lock`

     `mount_lock` is a global seqlock, a bit like `rename_lock`. It can be used to check if any change has been made to any mount points.

While walking down the tree (away from the root), this lock is used when crossing a mount point to check that the crossing was safe. That is, the value in the seqlock is read, then the code finds the mount that is mounted on the current directory, if there is one, and increments the `mnt_count`. Finally, the value in `mount_lock` is checked against the old value. If there is no change, then the crossing was safe. If there was a change, the `mnt_count` is decremented and the whole process is retried.

When walking up the tree (towards the root) by following a ".." link, a little more care is needed. In this case the seqlock (which contains both a counter and a spinlock) is fully locked to prevent any changes to any mount points while stepping up. This locking is needed to stabilize the link to the mounted-on dentry, which the refcount on the mount itself doesn't ensure.

RCU

     Finally the global (but extremely lightweight) RCU read lock is held from time to time to ensure certain data structures don't get freed unexpectedly. In particular, it is held while scanning chains in the dcache hash table, and the mount point hash table.

#### Bringing it together with `struct nameidata`

Throughout the process of walking a path, the current status is stored in a `[struct nameidata](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n494>)`, "namei" being the traditional name — dating all the way back to First Edition Unix — of the function that converts a "name" to an "inode". `struct nameidata` contains (among other fields):

`struct path path`

     A `path` contains a `struct vfsmount` (which is embedded in a `struct mount`) and a `struct dentry`. Together these record the current status of the walk. They start out referring to the starting point (the current working directory, the root directory, or some other directory identified by a file descriptor) and are updated on each step. A reference through `d_lockref` and `mnt_count` is always held.

`struct qstr last`

     This is a string together with a length (i.e. _not_ `nul` terminated) that is the "next" component in the pathname.

`int last_type`

     This is one of `LAST_NORM`, `LAST_ROOT`, `LAST_DOT`, `LAST_DOTDOT`, or `LAST_BIND`. The `last` field is only valid if the type is `LAST_NORM`. `LAST_BIND` is used when following a symlink and no components of the symlink have been processed yet. Others should be fairly self-explanatory.

`struct path root`

     This is used to hold a reference to the effective root of the filesystem. Often that reference won't be needed, so this field is only assigned the first time it is used, or when a non-standard root is requested. Keeping a reference in the `nameidata` ensures that only one root is in effect for the entire path walk, even if it races with a `chroot()` system call.

The root is needed when either of two conditions holds: (1) either the pathname or a symbolic link starts with a "`/`", or (2) a "`..`" component is being handled, since "`..`" from the root must always stay at the root. The value used is usually the current root directory of the calling process. An alternate root can be provided as when `sysctl()` calls `file_open_root()`, and when NFSv4 or Btrfs call `mount_subtree()`. In each case, a pathname is being looked up in a very specific part of the filesystem, and the lookup must not be allowed to escape that subtree. It works a bit like a local `chroot()`.

Ignoring the handling of symbolic links, we can now describe the `link_path_walk()` function, which handles the lookup of everything except the final component as:

> Given a path (`name`) and a `nameidata` structure (`nd`), check that the current directory has execute permission and then advance `name` over one component while updating `last_type` and `last`. If that was the final component, then return, otherwise call `walk_component()` and repeat from the top.

`walk_component()` is even easier. If the component is `LAST_DOTS`, it calls `handle_dots()`, which does the necessary locking as already described. If it finds a `LAST_NORM` component, it first calls `lookup_fast()`, which only looks in the dcache, but will ask the filesystem to revalidate the result if it is that sort of filesystem. If that doesn't get a good result, it calls `lookup_slow()`, which takes the `i_mutex`, rechecks the cache, and then asks the filesystem to find a definitive answer. Each of these will call `follow_managed()` (as described below) to handle any mount points.

In the absence of symbolic links, `walk_component()` creates a new `struct path` containing a counted reference to the new dentry and a reference to the new `vfsmount`, which is only counted if it is different from the previous `vfsmount`. It then calls `[path_to_nameidata()](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n814>)` to install the new `struct path` in the `struct nameidate` and drop the unneeded references.

This "hand-over-hand" sequencing of getting a reference to the new dentry before dropping the reference to the previous dentry may seem obvious, but is worth pointing out so that we will recognize its analogue in the "RCU-walk" version.

#### Handling the final component

`link_path_walk()` only walks as far as setting `nd->last` and `nd->last_type` to refer to the final component of the path. It does not call `walk_component()` that last time. Handling that final component remains for the caller to sort out. Those callers are `path_lookupat()`, `path_parentat()`, `path_mountpoint()`, and `path_openat()`, each of which handles the differing requirements of different system calls.

`[path_parentat()](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2144>)` is clearly the simplest; it just wraps a little bit of housekeeping around `link_path_walk()` and returns the parent directory and final component to the caller. The caller will be either aiming to create a name (via `filename_create()`) or remove or rename a name (in which case `user_path_parent()` is used). They will use `i_mutex` to exclude other changes while they validate and then perform their operation.

`[path_lookupat()](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2088>)` is nearly as simple; it is used when an existing object is wanted such as by `stat()` or `chmod()`. It essentially just calls `walk_component()` on the final component through a call to `lookup_last()`. `path_lookupat()` returns just the final dentry.

`[path_mountpoint()](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n2410>)` handles the special case of unmounting, which must not try to revalidate the mounted filesystem. It effectively contains, through a call to `mountpoint_last()`, an alternate implementation of `lookup_slow()`, which skips that step. This is important when unmounting a filesystem that is inaccessible, such as one provided by a dead NFS server.

Finally `[path_openat()](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n3270>)` is used for the `open()` system call; it contains, in support functions starting with `do_last()`, all the complexity needed to handle the different subtleties of `O_CREAT` (with or without `O_EXCL`), final "`/`" characters, and trailing symbolic links. We will revisit this in the final part of this series, which focuses on those symbolic links. `do_last()` will sometimes, but not always, take `i_mutex`, depending on what it finds.

Each of these, or the functions which call them, need to be alert to the possibility that the final component is not `LAST_NORM`. If the goal of the lookup is to create something, then any value for `last_type` other than `LAST_NORM` will result in an error. For example if `path_parentat()` reports `LAST_DOTDOT`, then the caller won't try to create that name. They also check for trailing slashes by testing `last.name[last.len]`. If there is any character beyond the final component, it must be a trailing slash.

#### Revalidation and automounts

Apart from symbolic links, there are only two parts of the "REF-walk" process not yet covered. One is the handling of stale cache entries and the other is automounts.

On filesystems that require it, the lookup routines will call the `->d_revalidate()` dentry method to ensure that the cached information is current. This will often confirm validity or update a few details from a server. In some cases it may find that there has been change further up the path and that something that was thought to be valid previously isn't really. When this happens the lookup of the whole path is aborted and retried with the `LOOKUP_REVAL` flag set. This forces revalidation to be more thorough. We will see more details of this retry process in the next article.

Automount points are locations in the filesystem where an attempt to lookup a name can trigger changes to how that lookup should be handled, in particular by mounting a filesystem there. These are covered in greater detail in `[autofs4.txt](<https://www.kernel.org/doc/Documentation/filesystems/autofs4.txt>)` in the Linux documentation tree, but a few notes specifically related to path lookup are in order here.

The Linux VFS has a concept of "managed" dentries which is reflected in function names such as `[follow_managed()](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/fs/namei.c?id=052b398a43a7de8c68c13e7fa05d6b3d16ce6801#n1138>)`. There are three potentially interesting things about these dentries corresponding to three different flags that might be set in `dentry->d_flags`: 

`DCACHE_MANAGE_TRANSIT`

     If this flag has been set, then the filesystem has requested that the `d_manage()` dentry operation be called before handling any possible mount point. This can perform two particular services:

  1. It can block to avoid races. If an automount point is being unmounted, the `d_manage()` function will usually wait for that process to complete before letting the new lookup proceed and possibly trigger a new automount.

  2. It can selectively allow only some processes to transit through a mount point. When a server process is managing automounts, it may need to access a directory without triggering normal automount processing. That server process can identify itself to the `autofs` filesystem, which will then give it a special pass through `d_manage()` by returning `-EISDIR`.

`DCACHE_MOUNTED`

     This flag is set on every dentry that is mounted on. As Linux supports multiple filesystem namespaces, it is possible that the dentry may not be mounted on in _this_ namespace, just in some other. So this flag is seen as a hint, not a promise. If this flag is set, and `d_manage()` didn't return `-EISDIR`, `lookup_mnt()` is called to examine the mount hash table (honoring the `mount_lock` described earlier) and possibly return a new `vfsmount` and a new `dentry` (both with counted references).

`DCACHE_NEED_AUTOMOUNT`

     If `d_manage()` allowed us to get this far, and `lookup_mnt()` didn't find a mount point, then this flag causes the `d_automount()` dentry operation to be called. 

The `d_automount()` operation can be arbitrarily complex and may communicate with server processes etc., but it should ultimately either report that there was an error, that there was nothing to mount, or should provide an updated `struct path` with new `dentry` and `vfsmount`. In the latter case, `finish_automount()` will be called to safely install the new mount point into the mount table.

There is no new locking of import here and it is important that no locks (only counted references) are held over this processing due to the very real possibility of extended delays. This will become more important next time when we examine RCU-walk which is particularly sensitive to delays.

#### To be continued

With a dcache and a mount table, slashes at the start and the end, final components handled multiple ways and with a variety of locking primitives; we have already examined, and hopefully understood, a fair degree of complexity. There is still more to come though. Next week we will build on this to explore the very different challenges that arise when we try to perform pathname lookup without taking any locks at all.

Finally, I'd like to make a note of thanks to Al Viro for reviewing an early draft of this article and highlighting a number of errors.
