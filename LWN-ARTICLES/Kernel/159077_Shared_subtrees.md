---
title: Shared subtrees
url: https://lwn.net/Articles/159077/
date: "November 8, 2005"
category: "Filesystems; Namespaces-Mount namespaces; Shared subtrees"
author: 
---

The shared subtrees patch set, written primarily by Ram Pai, has been in circulation for some time, but without a whole lot of discussion. Those patches have now been merged into the pre-2.6.15 mainline, so the time has come for a closer look. In short, shared subtrees allow a system administrator to configure, in great detail, how various filesystem mounts should appear in the tree, how they relate to each other, and how they propagate between namespaces. There are two motivations for this work: 

  * The "files as directories" feature of the reiser4 filesystem allows a user to create, via hard links, a directory which appears in multiple places in the filesystem. That feature has long been disabled due to the deadlock issues which it raised. Shared subtrees are a step toward implementing "files as directories" in a safe manner. 

  * The merging of the filesystems in user space patch, and some of the [permissions issues](<http://lwn.net/Articles/131856/>) associated with it, has increased the desire to be able to run users in their own filesystem namespaces. Per-user namespaces are currently awkward at best; shared subtrees will help make them easier to manage. 

It should be noted that the patches merged into the mainline are not a complete solution for either of the above problems, but they are a step in that direction. The per-user namespaces example will be used in what follows to illustrate how the various subtree options work. 

Every filesystem in Linux is mounted within a specific namespace. The kernel has long supported the creation of multiple namespaces, but, in most situations, that feature is not used. So the typical Linux system has a single namespace which is shared between all processes on the system. When separate namespaces are used, they are usually in the context of sandboxing and isolation. There would be advantages, however, to making more extensive use of namespaces. 

![\[simple tree\]](https://static.lwn.net/images/ns/kernel/subtrees1.png) Imagine, for starters, a simple filesystem hierarchy which looks something like the diagram at the right. Clearly, a few directories have been left out for simplicity. The only unusual thing is that a couple of directories have been created under `/subtree` for users "alice" and "bob". We would like to use those directories as the root for each user's own private view of the filesystem. 

The first step is to create a copy of the root filesystem under each user's subtree directory using bind mounts. The result of such an operation will look like the diagram below.  ![\[diagram\]](https://static.lwn.net/images/ns/kernel/subtrees2.png) Note that the `/subtree` tree has been bound into each user's namespace as well. This propagation cuts down on the isolation between users, since they can see each others' subtrees. As the number of users grows, it also complicates the namespaces considerably, as each set of subtrees must be replicated over and over. 

This loss of isolation and explosion of mount points can be avoided through the use of "unbindable" mounts, a new feature added by the sharable subtrees patch. Said mounts cannot be bound into other places, and will not be propagated into new subtrees. So the administrator could execute a series of commands like: 

```
mount --bind /subtree /subtree
        mount --make-unbindable /subtree
```

This incantation turns `/subtree` into a magic point which cannot be rebound. If, after this has been done, the administrator makes the per-user bind mounts of the root filesystem, the portion under `/subtree` will be pruned, with a result which looks like this: 

![\[diagram\]](https://static.lwn.net/images/ns/kernel/subtrees3.png)

Now imagine that the system administrator mounts a CDROM under `/mnt`. The result will look like: 

![\[diagram\]](https://static.lwn.net/images/ns/kernel/subtrees4.png)

Note that the CDROM mount is not visible in the per-user namespaces, so bob and alice will be unable to look at the contents of the CD. That might be the intended result, but imagine it's not, that the administrator wants all users to be able to see things mounted on `/mnt`. The answer is a "sharable" mount, one which is automatically propagated into every place where the original mount appears. So, the administrator need only perform another new incantation: 

```
mount --bind /mnt /mnt
        mount --make-shared /mnt
```

After this, `/mnt` is a sharable mount. Any changes made there will appear in any namespace where `/mnt` appears. The resulting tree would look something like this: 

![\[diagram\]](https://static.lwn.net/images/ns/kernel/subtrees5.png)

Many administrators might rather just make the entire filesystem tree sharable, rather than try to anticipate where changes could be made. If the root is made sharable in this way, any new filesystems which are mounted will propagate throughout the tree. This propagation works all ways; if alice mounts the CD within her subtree, it will still appear in all of the subtrees. 

Of course, this behavior might not always be desirable. If, for example, bob is using FUSE to mount an "ssh filesystem" from a remote host, he would prefer that this filesystem not be visible to other users at all. But bob would still like to see filesystems mounted elsewhere, and does not want to give up the advantages of a shared subtree. The answer is yet another type of mount, called a "slave" mount. Slave mounts are selfish: they remain tied to their parent mount, and receive new mounts from there. Anything mounted underneath the slave mount, however, will not be propagated elsewhere. So each user can have his or her own filesystems which are not part of the global hierarchy: 

![\[diagram\]](https://static.lwn.net/images/ns/kernel/subtrees6.png)

The shared subtrees patch also adds a "private" mount type, which is essentially how mounts in 2.6.14 and prior kernels work. A private mount will not be propagated to any other mounts, but it can (unlike an unbindable mount) be explicitly propagated via a bind operation. 

Internally, the patches create the concept of a "peer group," among which mount events are propagated. A new `mnt_share` field (a list of peers) has been added to the `vfsmount` structure for this purpose. A couple of other lists (`mnt_slave_list` and `mnt_slave`) have been added for keeping track of slave mount relationships. A new `MNT_UNBINDABLE` flag marks unbindable mounts. And, of course, a great deal of locking work has been done to make all of this work in a safe manner. Al Viro has worked with a few iterations of the shared subtrees patch, with the result that it is now considered to be ready for the mainline. 

The shared subtrees patch is a big step forward: it is a fundamental change to the virtual filesystem layer which greatly increases the flexibility in how namespaces can be populated and presented to users. What remains, at this point, is some work on the namespace side of things. Namespaces are still unnamed objects which can only be inherited from a parent process; there is no easy way to create and attach to a per-user namespace. Finishing the job will take some work, but, chances are, the hardest part of the problem has been solved. 

For more information, see [the extensive documentation file](<https://lwn.net/Articles/159092/>) shipped with the patch.
