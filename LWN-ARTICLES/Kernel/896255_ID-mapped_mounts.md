---
title: "ID-mapped mounts"
url: https://lwn.net/Articles/896255/
date: "May 30, 2022"
category: "Filesystems-Mounting; Namespaces-User namespaces"
author: "By Jake Edge May 30, 2022 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 30, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

The [ID-mapped mounts](<https://lwn.net/Articles/837566/>) feature was added to Linux in 5.12, but the general idea behind it goes back a fair bit further. There are a number of different situations where the user and group IDs for files on disk do not match the current human (or process) user of those files, so ID-mapped mounts provide a way to resolve that problem—without changing the files on disk. The developer of the feature, Christian Brauner, led a discussion at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM) on ID-mapped mounts. 

He began with an introduction. There are multiple use cases, but he likes to talk about portable home directories first because they are not related to containers, which many think is the sole reason for ID-mapped mounts. A portable home directory would be on some kind of removable media that can be attached to various systems, some of which have a different user and group ID for the user, but, of course, the media has fixed values for those IDs. ID-Mapped mounts allow the device to be mounted on the system with the IDs remapped to those of the user on the local system. 

[ ![\[Christian Brauner\]](https://static.lwn.net/images/2022/lsfmm-brauner-sm.png) ](<https://lwn.net/Articles/896513/>)

Beyond that, of course, are various container use cases, such as sharing a root filesystem with multiple containers, each of which is using its own user namespace with a different mapping for UID 0\. Each of the containers needs to be able to access the files as "root", but UID 0 inside the namespace is mapped to some nonzero UID on the host system; an ID-mapped mount would enable that nonzero ID to be mapped to UID 0 for filesystem access. Similarly, sharing data between a host filesystem and one in a user namespace may require remapping the IDs. Some of these cases were handled with expensive recursive `chown` calls before ID-mapped mounts came along. 

There are some filesystems that can be used in user-namespace-based containers, most notably overlayfs, but there are still lots of limitations and the main filesystem types, Btrfs, XFS, and ext4, are not really able to be used in that manner. Once all of the use cases were gathered, he said, the most flexible solution turned out to be a per-mount mapping of UIDs and GIDs, which is what ID-mapped mounts provide. 

The API for the feature uses the [`mount_setattr()`](<https://man7.org/linux/man-pages/man2/mount_setattr.2.html>) system call, which allows changing the ID mappings as well as other attributes of mounts. Brauner clarified that the feature applies to all virtual filesystem (VFS) mounts, so bind mounts are included. Unlike [`mount()`](<https://man7.org/linux/man-pages/man2/mount.2.html>), `mount_setattr()` allows changing mount attributes recursively. 

Using the feature requires passing a flag and a file descriptor to `mount_setattr()`; the file descriptor is that of a user namespace that does the ID mapping that should be applied to the mount. The implementation was done in the VFS layer, so individual filesystems "do not need to be really aware of it"; there are APIs available to make it easy on the filesystems, he said. Ted Ts'o asked about a command-line tool for doing an ID-mapped mount; Brauner said that one should be merged soon into [util-linux](<https://github.com/util-linux/util-linux>). 

Amir Goldstein noted that fstests already has a binary tool for testing these mounts. Brauner added that there are 15K lines of code in tests, already upstream in fstests, for ID-mapped mounts that aim to test the feature in all possible combinations. That includes things like access-control lists (ACLs), Linux capabilities, setuid and setgid execution, and so on. Every time a bug or regression is found, a new test is added to the suite. 

He spent a bit of time demonstrating the tool and the feature, noting that the mapping works in both directions: IDs of files in the mount follow the mapping and files created within the mount have the reverse-mapped IDs outside of it. The feature is already being used by various tools, such as systemd-nspawn and systemd-homed; it has also been added to the runC container specification, so "there is lots of activity going on around this". 

Currently, ext4, XFS, Btrfs, and several other filesystems support the feature; there is a patch set for overlayfs that is on-track to be merged soon. David Howells asked what filesystems need to do to support ID-mapped mounts. Brauner said that "in principle it is easy" to do so. Network filesystems may have some additional wrinkles, however; he has a patch set for Ceph but it still needs more work. The changes for ext4 and XFS were small, he said, and others are likely to be similar because most filesystems do not really use the IDs directly. The XFS quota-handling code does use the IDs, so it needed a bit more work. There is a [long document](<https://www.kernel.org/doc/html/latest/filesystems/idmappings.html>) available and he is willing to help add it to other filesystems. 

Network filesystems need to determine which ID they want to send to the server, he said. Normally, the mapped ID is the right choice, but that may not be true for all cases. 

Chuck Lever asked how the ID mapping could be changed for an existing mount and wondered if it could just be remounted to make that change. Brauner said that no changes are allowed once the namespace has been attached to the mount or the mount has been attached to the filesystem. Due to "lifetime issues" with regard to the use of the mapping, it is too complicated to allow changes once the filesystem has been fully mounted. Using the new mount API, a user will create a detached mount, then set the ID mapping on it, then, finally, attach it to the filesystem. 

Lever also asked about the limits for the number of entries in the mapping; for example, in a system with thousands of users, where each user should be mapped to their own ID in a single mount. Brauner said that user namespaces were originally limited to five mappings, but he raised that limit to 340 in 2015 or 2016. It will be difficult to increase it beyond that, he said, because mapping is done in a hot path; he optimized the data structure for the mappings and increasing it further will have a performance impact. 

Ts'o wondered if there was any thinking about supporting "project IDs", which are used by some container systems; those IDs are used for project-wide quotas in filesystems. Brauner said that project ID needs to be revisited, since "we have dodged this issue for years". The intended semantics are not clear, so he has been confused when looking into it. 

While both XFS and ext4 support those IDs, Ts'o said he is confused by the semantics as well, at least with respect to user namespaces. He and Darrick Wong discussed it at one point and it was not clear whether both filesystems worked the same way, though there is an intention to unify their behavior. Brauner said that quota handling is not the same between different filesystems in Linux; each seems to have its own quirks. In the Zoom chat, Jan Kara pointed out that ID-mapping changes had not been made to the VFS quota code, at least yet; that was relayed as time expired on the session, however.
