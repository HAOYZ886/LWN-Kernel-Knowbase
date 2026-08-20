---
title: Lazy file reflink
url: https://lwn.net/Articles/789038/
date: "May 22, 2019"
category: Filesystems
author: "By Jake Edge May 22, 2019 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
May 22, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Amir Goldstein has a use case for a feature that could be called a "lazy file reflink", he said, though it might also be described as "VFS-level snapshots". He went through the use case, looking for suggestions, in a session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). He has already implemented parts of the solution, but would like to get something upstream, which would mean shifting from the stacked-filesystem approach he has taken so far. 

He has a working prototype of some of what he wants, which he presented two years ago as [overlayfs snapshots](<https://lwn.net/Articles/719772/>). It has improved since then. The idea was to identify a subdirectory and snapshot it, so that any changes to the files in that hierarchy would be handled in a copy-on-write (CoW) fashion. It was done at the VFS layer, so it did not matter what actual filesystem type was being used. It worked using [`FICLONE`](<http://man7.org/linux/man-pages/man2/ioctl_ficlonerange.2.html>) operations or by making file copies for file changes. That means you would want to use it on filesystems that support clone/reflink operations, though filesystems that support their own snapshots, such as Btrfs, probably are not good candidates. 

His company is using the VFS snapshot mechanism, but only to track namespace changes: file renames, new files, and deleted files. It is not using the mechanism for tracking changes to the file data, which is convenient because that means it does not need the underlying filesystem to support clone operations. 

Instead, for changes to the file data, he is using the [filesystem change journal](<https://lwn.net/Articles/755277/>) that he talked about at last year's LSFMM. This is similar to the [change journal](<https://docs.microsoft.com/en-us/windows/desktop/FileIO/change-journals>) available with NTFS; it does persistent change tracking in a way that is reliable, unlike solutions based on fsnotify, which underlies [inotify](<http://man7.org/linux/man-pages/man7/inotify.7.html>) and [fanotify](<http://man7.org/linux/man-pages/man7/fanotify.7.html>). Fsnotify can lose events if there is an overflow or crash. The change journal guarantees that changes in a particular directory will be seen. 

He has this code running in production and the [code is public](<https://github.com/amir73il/overlayfs/wiki/Overlayfs-snapshots>), but he would like to make it more widely usable. There are some limitations since it is implemented as a stacked filesystem. There are other use cases, such as [Watchman](<https://facebook.github.io/watchman/>) from Facebook and [VFS for Git](<https://github.com/microsoft/VFSForGit>) from Microsoft; both are trying to solve similar problems. Watchman is using inotify recursively with all of the disadvantages that come with that. 

So he would like to provide a way for applications to watch changes on, say, a Git project, and to do it consistently and reliably without using a stacked filesystem. There are two gaps that he has identified; he is looking for ideas on how to fill them. The first is that the hooks he has available only allow getting events when a file is opened for write. If it is already open, there is no facility to get a notification on the first time it is modified via a `write()` or a change to a region mapped with `mmap()`. He would like to be able to freeze the file, flush its pages to persistent storage, then get an event when the first write happens after that. He would like to implement that in a non-intrusive way. 

The second gap is the lack of a way to do subtree filtering at the kernel level. That way, a watch could be established on a subtree and only events from that subtree would be reported; macOS has this facility. His thinking is to have an API to mark a directory as a subtree root, then perhaps something could be added to the VFS to directly handle subtrees. There may be some commonality with some gaps that Btrfs has for subvolume handling, he said. It would provide the ability to create fixed subtrees that users cannot change. 

Jan Kara said that for fanotify and things like it, he does not think isolating a subtree so that users cannot, for example, hard link into or out of them is what is needed. Goldstein said that one of his ideas was that you could not rename files into or out of the subtree, but Kara said that would have strange semantics that would not be understandable for user-space programs. 

There was some discussion on how the subtree support could be implemented, but the assembled developers did not seem to entirely grasp what Goldstein was envisioning—or perhaps it was only me who did not follow what he was after. In any case, Goldstein said that he would be trying to implement something that he could post for comment. He asked if attendees had thoughts on the first problem he posed: getting a pre-write notification on an open file. Prior to LSFMM, he had [summarized](<https://lwn.net/ml/linux-fsdevel/CAOQ4uxgq3iR%2Bw%3DqTrbA6Pqney4NjJP7fAQpWVBSbrZYS%2BiC4cg%40mail.gmail.com/>) his ideas in a post to the linux-fsdevel mailing list. 

Goldstein noted that when he posted his [initial request](<https://lwn.net/ml/linux-fsdevel/CAOQ4uxgqm-m1Zj073o_vSnwkTbGObJiQ-CdWV2ESd_P-29=jZw@mail.gmail.com/>) for an LSFMM slot on the topic, Dave Chinner had [replied](<https://lwn.net/ml/linux-fsdevel/20190128212642.GQ4205@dastard/>) with some thoughts on a per-file freeze API, so he may have another use case. What Goldstein is looking for is different than a mandatory lock on a file because others processes could still have the file open for write. Like a filesystem freeze, though, write operations would not complete until the unfreeze (or, in his case, the notification is acknowledged). Ted Ts'o asked if what he wanted was a way to make any attempts to modify the file block, while reads could still complete. Goldstein said that what he needs is a notification on the first change to a file after a given point in time. 

That notification needs to be given before the file changes so that the change journal can record it persistently. In fsnotify terms, what he wants would be a write pre-modification one-shot mark, Kara said. Ts'o asked if he was asking for user space to be able to get the notification and acknowledge it before the write could proceed. Goldstein said that he did not need the user-space side of that, since his use case is inside the kernel, but other use cases might want that capability. 

Ts'o asked if any modification to the page cache for the file might need to send this notification, which could actually stop the change from happening. It could be done with a new security hook, Goldstein said; there is currently no security hook for writes to `mmap()` regions. He is not suggesting a security hook for every page fault, but does want to block the first modification until it gets recorded; if the notification does not get acknowledged, then the application would get a segmentation fault. 

There are concerns about doing this kind of thing from the page-fault-handling code. Goldstein only wants the first write to any page for a given inode to trigger his notification, but if it were a security hook, others could use it differently, which might result in page faults being arbitrarily delayed. Kara noted that currently the security hooks are always called from a system-call context, while this would be called from the page-fault context, which is significantly different, especially with regard to locking. 

Overall, the consensus seemed to be that this would be complex and difficult to implement correctly. There were problems implementing the security hook for `open()`, Ts'o said, and this will "be ten times worse".
