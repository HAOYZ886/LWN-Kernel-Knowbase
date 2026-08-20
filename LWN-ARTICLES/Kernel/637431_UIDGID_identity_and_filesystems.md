---
title: UID/GID identity and filesystems
url: https://lwn.net/Articles/637431/
date: "March 23, 2015"
category: Filesystems
author: "By Jake Edge March 23, 2015 LSFMM 2015"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
March 23, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

"User namespaces only solve half the problem", Andy Lutomirski said to start off his session at the 2015 LSFMM Summit. User namespaces remap user IDs (UIDs) and group IDs (GIDs) in the running kernel, but they don't do anything for the UID and GID values stored in filesystems. Those IDs are simply integers stored in the filesystem metadata. 

Lutomirski noted that when inserting a USB stick with a "real filesystem, not FAT" on it, the mounted filesystem will have UIDs and GIDs that are likely to be wrong. It would be nice, he said, if instead the files showed up as being owned by the user's UID. 

This is also a problem for both NFS and FUSE filesystems, he continued. There is a partial solution in that mounting a FUSE filesystem inside a user namespace will map the UIDs inside the namespace before writing them to the filesystem. NFS has a solution as well. He wondered if there could be a more general approach. 

Dave Chinner pointed out that some filesystems have mount options to do simple UID remapping. Those options might simply squash all UID/GIDs on the filesystem into a single UID/GID. An option like that could be added to the virtual filesystem (VFS) layer so that all filesystems had access to it. 

That might be a reasonable way to approach the problem, Lutomirski said. Obviously NFS has already solved it, he said, though he had not looked to see what it does. Jeff Layton said that NFS has traditionally mapped UIDs and GIDs between the server and the client. That was originally done using strings for the user and group names, which would get mapped at the other end to integers. The current NFS solution is more complicated, Bruce Fields said, involving LDAP lookups, which is probably not what Lutomirski is looking for. 

For his use case, squashing to a single UID would be sufficient, Lutomirski said. Handling Linux Security Module (LSM) contexts is trickier, but that could perhaps be added later. There was some discussion of the different ways that filesystems interpret the `uid=` and `gid=` mount options; he would like to see there be some uniformity, which would might require an entirely new mount option (possibly something like `vfs_uid=`). 

[I would like to thank the Linux Foundation for travel support to Boston for the summit.]
