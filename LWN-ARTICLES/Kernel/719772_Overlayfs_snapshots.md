---
title: Overlayfs snapshots
url: https://lwn.net/Articles/719772/
date: "April 12, 2017"
category: Overlayfs
author: "By Jake Edge April 12, 2017 Vault"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
April 12, 2017

* * *

[Vault](<https://lwn.net/Archives/ConferenceByYear/#2017-Vault>)

At the [2017 Vault](<http://events.linuxfoundation.org/events/vault>) storage conference, Amir Goldstein gave a talk about using [overlayfs](<https://lwn.net/Articles/403012/>) in a novel way to create snapshots for the underlying filesystem. His company, CTERA Networks, has used the [NEXT3](<http://next3.sourceforge.net/>) ext3-based filesystem with snapshots, but customers want to be able to use larger filesystems than those supported by ext3. Thus he turned to overlayfs as a way to add snapshots for XFS and other local filesystems. 

NEXT3 has a number of shortcomings that he wanted to address with overlayfs snapshots. Though it only had a few requirements, which were reasonably well supported, NEXT3 never got upstream. It was ported to ext4, but his employer stuck with the original ext3-based system, so the ext4 version was never really pushed for upstream inclusion. 

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2017/vault-goldstein-sm.jpg) ](<https://lwn.net/Articles/719789/>)

One of the goals of the [overlayfs snapshots](<https://lwn.net/Articles/708370/>) (ovfs-snap) project is for it to be included upstream for better maintainability. It will also allow snapshots at a directory subtree level; the alternative mechanisms for snapshots, Btrfs or LVM thin provisioning (thinp), are volume-level snapshots. Those two also allow writable and nested snapshots, while ovfs-snap does not. The "champion feature" for ovfs-snap is that the inode and page cache are shared, which is not true of the others. For a large number of containers, it becomes inefficient to have multiple copies of the same data in these caches, he said. 

Goldstein then moved into a demonstration of the feature. In previous versions of the talk, he did the demo at the end but, based on some feedback, has moved it near the beginning. It is a bit hard to describe, however, as with many demos. The basic idea is that snapshot mounts turn overlayfs on its head: the lower layer, which normally doesn't change in a normal overlayfs mount, is allowed to change, while the upper layer is modified to cover up the changes made in the lower so that the upper has the same contents as the lower at the time of the snapshot. 

This is done using a special "snapshot mount" that is a shim over the lower layer to intercept filesystem operations to it. Before those operations are performed, the current state of the lower layer is copied up to the upper layer. The upper layer is a snapshot overlay (which is different from a snapshot mount) that effectively records the state of the lower layer before any changes are made to it. 

So the lower layer must be accessed through the snapshot mount, but the upper layer is simply a regular overlayfs that can be accessed as usual to get a view of the filesystem at the time of the snapshot. Multiple snapshots can be supported by having a lower layer shared between multiple upper layers, each of which hides any changes made to the lower after they were mounted (which is when the snapshot is taken). 

These snapshots can work for any size of directory tree. It will also work on top of Btrfs, XFS, or another local filesystem. The upper layer will record what has changed, but at the file level, not at the block level. One consequence of this design is that changing one byte of a large file results in a copy-up operation for the whole file. In addition, currently only one copy-up at a time is supported, so a large copy-up blocks any others. 

Some new features are coming that will address some of these problems. For the container use case, Goldstein said, the copy-up performance issue is not usually a real problem. But for his use case, with large XFS files, copy-up performance is important. So, for 4.10, a "clone up" operation was added when the underlying filesystem supports the clone operation (as XFS and others do). The clone will do a copy-on-write "copy" of the file before it is modified so that only changed blocks actually get copied. There is also support for doing concurrent copy-up operations that is coming in 4.11. 

Goldstein presented a couple of different use cases for ovfs-snap. For a short-lived snapshot for backup purposes, an ovfs-snap provides a file-level copy-on-write filesystem. Changes to the lower layer trigger the copy-up so the snapshot is consistent with the state at the time of the backup. The lower layer can be accessed at near-native performance, while accessing the snapshot can tolerate some lesser performance, he said. 

One could also use ovfs-snap to allow access to multiple previous versions of the filesystem. Multiple upper layers can be composed to create a view of the filesystem at any of the snapshot times, while the lower layer remains mounted and accessible. Those snapshots are read-only, however, unlike Btrfs or LVM thinp snapshots. 

The rules for maintaining an overlay that represents a snapshot are fairly straightforward. Files must copied (or cloned) up before they are modified or deleted in the lower layers. A whiteout marking a deletion must be added before a file gets created in the lower layer. A directory in the snapshot overlay must be redirected when a directory in the lower layer gets renamed. Finally, when a lower layer directory gets deleted, an opaque directory must be created in the snapshot. 

Taking a snapshot is a somewhat complicated process (see slide 15 in Goldstein's [slides [PDF]](<http://events.linuxfoundation.org/sites/events/files/slides/overlayfs-snapshots-vault-2017_0.pdf>) for more information). Simplifying that process is on the to-do list for the project. There are also plans to support merging snapshots as well as working on getting the code upstream. He finished the talk with the inevitable invitation to help work on the project; he pointed those interested at the [project wiki](<https://github.com/amir73il/overlayfs/wiki>). 

[I would like to thank the Linux Foundation for travel assistance to Cambridge, MA for Vault.]
