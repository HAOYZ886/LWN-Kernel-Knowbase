---
title: "Filesystem UID mapping for user namespaces: yet another shiftfs"
url: https://lwn.net/Articles/812504/
date: "February 17, 2020"
category: "Filesystems-shiftfs; Namespaces-User namespaces"
author: "By Jonathan Corbet February 17, 2020"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
February 17, 2020

The idea of an ID-shifting virtual filesystem that would remap user and group IDs before passing requests through to an underlying real filesystem has been around for a few years but has never made it into the mainline. Implementations have taken the form of [shiftfs](<https://lwn.net/Articles/718639/>) and [shifting bind mounts](<https://lwn.net/Articles/809125/>). Now there is [yet another approach to the problem](<https://lwn.net/ml/linux-kernel/20200211165753.356508-1-christian.brauner@ubuntu.com/>) under consideration; this one involves a theoretically simpler approach that makes almost no changes to the kernel's filesystem layer at all. 

ID-shifting filesystems are meant to be used with [user namespaces](<https://lwn.net/Articles/532593/>), which have a number of interesting characteristics; one of those is that there is a mapping between user IDs within the namespace and those outside of it. Normally this mapping is set up so that processes can run as root within the namespace without giving them root access on the system as a whole. A user namespace could be configured so that ID zero inside maps to ID 10000 outside, for example; ranges of IDs can be set up in this way, so that ID 20 inside would be 10020 outside. User namespaces thus perform a type of ID shifting now. 

In systems where user namespaces are in use, it is common to set them up to use non-overlapping ranges of IDs as a way of providing isolation between containers. But often complete isolation is not desired. James Bottomley's motivation for creating shiftfs was to allow processes within a user namespace to have root access to a specific filesystem. With the current patch set, instead, author Christian Brauner describes a use case where multiple containers have access to a shared filesystem and need to be able to access that filesystem with the same user and group IDs. Either way, the point is to be able to set up a mapping for user and group IDs that differs from the mapping established in the namespace itself. 

Shiftfs was a virtual filesystem that would pass operations through to an underlying filesystem while remapping (by applying a constant offset) the user and group IDs involved. The later bind-mount implementation did away with the separate filesystem and made the shifting a property of the mount itself. Brauner's approach, apparently sketched out at the 2019 Linux Plumbers Conference, is different; it makes the shifting a property of the user namespace itself. 

Processes in Linux, as in any Unix-like system, have associated user and group IDs. It is tempting to think that these IDs control access to files, but that is not quite true; instead, Linux maintains a separate user and group ID for filesystem access. These IDs can be changed (by an appropriately privileged process) using the [`setfsuid()`](<http://man7.org/linux/man-pages/man2/setfsuid.2.html>) and [`setfsgid()`](<http://man7.org/linux/man-pages/man2/setfsgid.2.html>) system calls. This feature is rarely used, so the filesystem user and group IDs are normally the same as the regular IDs, but the mechanism to separate the two sets of IDs has been there since nearly the beginning. 

The implementation of user namespaces necessarily understands these filesystem IDs (FSIDs), but that understanding has never been exposed outside the kernel. Brauner's patch set works by making the FSIDs visible and explicit, allowing them to be mapped independently of the normal IDs. In particular, it creates two new files (`fsuid_map` and `fsgid_map`) under the `/proc` directory for each process running inside a user namespace. These behave like the existing `uid_map` and `gid_map` files, in that they accept one or more ranges of IDs to remap, but they affect the FSIDs instead. 

So, for example, a system administrator can, on current systems, map 100 user IDs starting at zero inside the container to the range 10,000-10,100 outside by writing this line to `uid_map`: 

```
0 10000 100
```

By default, this mapping will also affect that namespace's FSIDs. But if the FSIDs should be mapped differently, say to a range starting at 20,000, then the administrator could write this to `fsuid_map`: 

```
0 20000 100
```

This mechanism is conceptually simpler than the ideas that came before, though it still requires a 24-part patch series to implement. It keeps all of the ID mapping in the same place and doesn't require special filesystem or mount types. So there is definitely something to like here. 

There is, though, a significant limitation in this implementation: the FSID mappings are global, and affect all of a container's filesystem activity, regardless of which filesystem is being accessed. The shiftfs or bind-mount approaches, instead, can be set up on a per-filesystem basis. Whether this loss of flexibility matters will depend on the specific use case in question; it seems likely that some users will want the ability to configure access to different filesystems differently. Adding that ability by way of the FSID mechanism may well be a complex task. 

Thus far, though, no potential users have spoken up to request this capability. This patch set is young, with the [second revision](<https://lwn.net/ml/linux-kernel/20200214183554.1133805-1-christian.brauner@ubuntu.com/>) having only just been posted, so it's possible that many users with an interest in this area have not yet encountered it. The third time might be the charm for this sort of ID-shifting capability, but to assume that to be the case would be premature.
