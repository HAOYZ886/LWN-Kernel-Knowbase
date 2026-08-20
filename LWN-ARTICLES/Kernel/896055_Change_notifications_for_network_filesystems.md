---
title: Change notifications for network filesystems
url: https://lwn.net/Articles/896055/
date: "May 25, 2022"
category: "Filesystems-Network; Inotify; fanotify"
author: "By Jake Edge May 25, 2022 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
May 25, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

Steve French led a discussion on change notifications for network filesystems in a session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). He is part of the Samba team and noted that both Windows and macOS clients get notified of new and changed files in a shared directory immediately, while on Linux that does not happen. He wanted to explore what it would take to add that functionality. 

On Windows and macOS, a file browser automatically shows changes to files in shared network filesystems, but at some point that broke for Linux clients. The inotify mechanism (and its predecessor, dnotify) were added to the kernel to support the Samba server, he said. Remote systems that are talking to a Samba server on Linux can see those kinds of changes, but remote Linux clients cannot. 

The client API changed at some point so that network filesystems have no easy way to register to receive these kinds of events. For SMB, he added an `ioctl()` command that can be used wait on notifications of these changes. But in order to use that, all of the client programs would need to change to make a filesystem-specific call in to get that information. 

[ ![\[Steve French\]](https://static.lwn.net/images/2022/lsfmm-french-sm.png) ](<https://lwn.net/Articles/896154/>)

The underlying problem is that the filesystem servers are not told that a Linux client wants to be notified of changes. That means Linux file browsers do not have the functionality that Windows and Mac users have come to expect. The inotify functionality does not have a hook into Ceph, AFS, or SMB to make them aware that a client wants notifications, he said. Chuck Lever noted that NFS has the notification capability in the protocol, but, like the others, it is not implemented for Linux. 

There is also the fanotify API, French said, but he does not know if it would be useful for what he is looking for. Amir Goldstein said that fanotify was originally created by antivirus vendors but that, more recently, work has gone into it to add more functionality. As of about Linux 5.10, fanotify provides almost a superset of the inotify functionality. 

One big feature that inotify lacks has been implemented in fanotify: watching an entire filesystem. There are not many applications that use it, because it is new, Goldstein said. He has added fanotify support to [inotify-tools](<https://github.com/inotify-tools/inotify-tools>) and its library, so there are now user-space tools that can be used to watch a filesystem or set of files using the fanotify API. 

There are many types of events that an SMB client can get from the server to tell it about changes to timestamps, file creation, file name changes, file deletion, and so on, French said. Those all seem to map reasonably well to fanotify/inotify events; changes to the access-control lists (ACLs) is not supported but might need to be, he said. Goldstein said that if there is enough interest, event types can be added to fanotify. 

On Linux, David Howells said, the file notifications are mostly used by desktop file managers. KDE starts a daemon to monitor changes and GNOME does something similar, he said; if notifications are not available, then they poll for the information. Goldstein said that it is not that notifications are not available, just that they are not granular enough and that there may be some kinds of changes that do not have notification events, so polling is used for those cases. 

Goldstein said that French had been asking for this feature for a long time. The FUSE developers "took a shot at implementing something", he said; it [added inotify support for virtiofs](<https://lwn.net/ml/linux-fsdevel/20211025204634.2517-1-iangelak@redhat.com/>). On the Zoom link, Vivek Goyal, who was involved in that work, said that inotify was chosen because it is simpler than fanotify. Whatever notification watches are placed on the local file are forwarded to the remote file server, which sets up inotify and forwards events back to the local filesystem. Based on the feedback on those patches, Goyal said, he has been trying to rework the patches to use fanotify but ran into a number of difficulties. There may be more limitations when using fanotify. French said that it is important to get a handle on what exactly can be supported because the alternative is "really painful": polling. 

Jan Kara, also via Zoom, said that it should be fairly straightforward to add the hook for filesystems to inform them that a watch has been added; in the simplest case, the filesystem just says that it does not support the feature. The more difficult part is that when the filesystem receives an event and wants to get it to the client filesystem in a way that user space can receive it via fanotify or inotify. For inotify, the inode number and file name are available to send to the client, but that is not true for fanotify, where you may only have the inode number. Goyal agreed that was the problem for virtiofs. 

The important thing is to provide a generic mechanism for filesystems so that applications do not have to use multiple filesystem-specific interfaces to get this information, French said. He also wants to avoid polling, which is particularly expensive when done across the network. Josef Bacik said that it seemed reasonable to add the hook to let the filesystems know when a watch has been added; it is up to French and Goyal to work out the details on that. 

Howells asked about subtree watches; on Windows you can get notified for changes within a subtree. He wondered if fanotify could add support for that. Goldstein said that it is something that everyone wants, but it is not trivial to do; several attempts have been made over the years, but nothing has been added. 

French said that the feature he is looking for is an asynchronous, non-perfect mechanism. Some filesystems, such as SMB and NFS, have strict approaches using delegations or leases to ensure that all events are seen, but that is not usually worth the cost. Those could be used to implement these change notifications, but it should be left up to the filesystem to decide that, he said. 

As time wound down, French also wanted to mention that he had not seen any tests for inotify and fanotify in xfstests (which are being renamed to "fstests"). It will be important to have tests to ensure that nothing breaks when the remote notifications are added. But Goldstein said that the tests for notifications are part of the Linux Test Project (LTP) tests. There is a test there for every new feature and regression tests for bugs that have been fixed. Ted Ts'o said that xfstests have historically been used by the developers of different filesystems, while features that were implemented in the virtual filesystem (VFS) layer were tested in LTP. That may need to change as the network filesystems add features to support notifications.
