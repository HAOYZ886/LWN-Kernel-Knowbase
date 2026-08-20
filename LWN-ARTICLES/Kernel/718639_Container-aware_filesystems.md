---
title: "Container-aware filesystems"
url: https://lwn.net/Articles/718639/
date: "April 3, 2017"
category: "Containers; Filesystems-shiftfs"
author: "By Jake Edge April 3, 2017 LSFMM 2017"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 3, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

We are getting closer to being able to do unprivileged mounts inside containers, but there are still some pieces that do not work well in that scenario. In particular, the user IDs (and group IDs) that are embedded into filesystem images are problematic for this use case. James Bottomley led a discussion on the problem in a session at the 2017 Linux Storage, Filesystem, and Memory-Management Summit. 

The various containerization solutions in Linux (Docker, LXC, rkt, etc.) all use the same container interfaces, he said. That leads to people pulling in different directions for different use cases. But the problem with UIDs stored in filesystem images affects all of them. These images are typically full root filesystems for the containers that have lots of files owned by the root user. 

[ ![\[James Bottomley\]](https://static.lwn.net/images/2017/lsfmm-bottomley-sm.jpg) ](<https://lwn.net/Articles/718642/>)

Bottomley has [proposed shiftfs](<https://lkml.org/lkml/2017/2/20/655>) as a potential solution to this problem. It is similar to a bind mount, but translates the filesystem UIDs based on the user namespace mapping. It can be used by unprivileged containers to mount a subtree that has been specifically marked by the administrator as being shiftfs-mountable. 

An earlier effort to solve the problem added the `s_userns` field to the superblock in order to do UID translations, but that is a per-superblock solution that does not work well for containers that want to share a specific mounted filesystem among containers with different UID mappings. With shiftfs, an inode operation will translate the UID based on the namespace mapping to that of the underlying filesystem before passing the operation the lower level. That means the virtual filesystem (VFS) does not need changes, which makes for a cleaner solution, Bottomley said. 

There are some significant security implications to allowing arbitrary directory trees to be shift-mounted in unprivileged containers, including the ability for users to create setuid-root binaries. So the administrator must mark those subtrees (using extended attributes in his prototype) that are safe to be mounted that way. 

Al Viro asked if there is a plan to allow mounting hand-crafted XFS or ext4 filesystem images. That is an easy way for an attacker to run their own code in ring 0, he said. The filesystems are not written to expect that kind of (ab)use. When asked if it really was that easy to crash the kernel with a hand-crafted filesystem image, Viro said: "is water wet?" 

Amir Goldstein said that the current mechanism is to use FUSE to mount the filesystems in the unprivileged containers. But Bottomley is concerned that the FUSE daemon can be exploited, so it should run in the unprivileged container as well. If you restrict the mounts to USB sticks, it means an attacker would need physical access, which has plenty of other paths for system compromise so it is "safe" in that sense. But if loopback mounting of filesystems is to be supported at some point, the filesystem code will need to have no exploitable bugs. 

In something of an aside, Goldstein reminded filesystem developers that their filesystems may be running under [overlayfs](<https://lwn.net/Articles/403012/>). He suggested that there needs to be more testing of different filesystems underneath overlayfs. 

While the attendees recognized the problem for unprivileged containers, there does not seem to be a consensus on the right route to take to solve it.
