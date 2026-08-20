---
title: Superblock watch for fsnotify
url: https://lwn.net/Articles/718802/
date: "April 5, 2017"
category: fanotify
author: "By Jake Edge April 5, 2017 LSFMM 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
April 5, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

At the 2017 Linux Storage, Filesystem, and Memory-Management Summit, Amir Goldstein led a discussion about the fsnotify filesystem notification subsystem and some [changes](<https://lwn.net/Articles/716973/>) he would like to see. Unfortunately, due to a bit of confusion of where the session would be held, I missed half of it; here's what I can reconstruct from the second half. Fsnotify is the internal kernel support subsystem for all three of the file notification APIs ([dnotify, inotify](<https://lwn.net/Articles/604686/>), and [fanotify](<https://lwn.net/Articles/339399/>)). 

Goldstein is trying to make fsnotify more scalable for getting notifications of changes in a large filesystem. To that end, he has proposed a "superblock watch" mechanism to efficiently report all changes made to a filesystem. For his use case, he just needs to be able to receive notifications when any file in any directory in the filesystem has changed (been created, deleted, or moved). There was a question about whether the names of the files that are changed should be included in the event, but Goldstein said he did not need that functionality (though others might); his application simply rescans the directory if anything has changed in it. 

Al Viro was concerned that the file names would not stay valid while notifications were being delivered. Jan Kara said that there could be races that would make it hard to reproduce the sequence of changes that were made to the directory. But adding names to the fsnotify events does add significant complexity to the code. There is a clear demand for being able to get notification events on a large directory tree, however, Kara said. For now, he is not convinced that adding file names into the event is warranted and it could lead to various kinds of problems. 

Goldstein said that the superblock watch is the simplest approach, rather than having a recursive fanotify watch on the mount point, which does not scale well. That API could eventually be extended to allow the creation of a [change journal](<https://msdn.microsoft.com/en-us/library/windows/desktop/aa363798\(v=vs.85\).aspx>) like NTFS supports, he said. There did not seem to be any fundamental opposition to the superblock watch feature as it stands.
