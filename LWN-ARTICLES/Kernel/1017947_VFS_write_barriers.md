---
title: VFS write barriers
url: https://lwn.net/Articles/1017947/
date: "April 23, 2025"
category: Filesystems
author: "By Jake Edge April 23, 2025 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 23, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

In the filesystem track at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF), Amir Goldstein wanted to resume discussing a feature that he had briefly introduced at the end of a [2023 summit session](<https://lwn.net/Articles/932415/>): filesystem "write barriers". The idea is to have an operation that would wait for any in-flight [`write()`](<https://www.man7.org/linux/man-pages/man2/write.2.html>) system calls, but not block any new `write()` calls as bigger hammers, such as freezing the filesystem, would do. His prototype implementation is used by a [hierarchical storage management](<https://en.wikipedia.org/wiki/Hierarchical_storage_management>) (HSM) system to create a crash-consistent change log, but there may be other use cases to consider. He [wanted to discuss](<https://lwn.net/ml/all/CAOQ4uxj00D_fP3nRUBjAry6vwUCNjYuUpCZg2Uc8hwMk6n%2B2HA%40mail.gmail.com/>) implementation options and the possibility of providing an API for user-space applications. 

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2025/lsfmb-goldstein-sm.png) ](<https://lwn.net/Articles/1018339/>)

Goldstein began by saying that he is working on a persistent change journal that is similar to what NTFS and Lustre provide; it records file changes, like events from [inotify](<https://www.man7.org/linux/man-pages/man7/inotify.7.html>) and [fanotify](<https://www.man7.org/linux/man-pages/man7/fanotify.7.html>) provide, but they are written to disk so that they survive a reboot or crash. He would like to work on a generic solution that is independent of any specific filesystem. He implemented the feature several years ago, as [overlayfs watch](<https://github.com/amir73il/overlayfs/wiki/Overlayfs-watch>), which his employer is using in production; overlayfs is not tied to a specific filesystem for its lower layers, so the feature is filesystem-independent. He never thought it made sense to get overlayfs watch into the upstream kernel, but that could perhaps be an option as well. 

It came later in the session, after a question from Jeff Layton, that Goldstein's use case is for users with a ""very large data set"" who need cloud synchronization, replication, and other features. The typical application wants to be able to do something, ""it doesn't matter what"", whenever there is a change to the filesystem; his company's application uses it to sync data to and from the cloud. In his application, the change journal records information to ensure that all filesystem changes are noted; it is used to avoid scanning the whole filesystem when the system is rebooted. 

He wants to make it possible to create the change journal from user space, perhaps implemented as a library that can provide the facility for any filesystem. There are a few requirements for such a journal, he said. It is meant to track changes, such as when a file is created in a directory. User space needs to get a blocking notification (which blocks further progress on the operation until user space responds) before the change is written to disk, as a kind of "intent to change" event, so that it can be written to the journal. After that, there needs to be an event indicating that the change has been done, which consumers can only see after the requisite filesystem changes have been made. He would like to do this with a VFS API so that it is independent of the filesystem type. 

His first proposed mechanism is with two new fanotify events. The first is `FAN_PRE_MODIFY` that is the "intent to change" event; it has sufficient information for recording the change in the journal. It is rare that only a single change is being done, he said; typically, there are multiple changes, to both metadata and file data, that are all signaled with the event. Once the changes are actually made, the "intent to change" has been completed and gets signaled with a `FAN_POST_MODIFY` event; then any consumer can read the changed data. 

Part of the information that gets sent with the event is a sequence number, which is needed so that the server can match up the events. Multiple threads could be making changes affecting the same directories, so a sequence number is needed to distinguish pairs of events. 

A different way to implement the feature was suggested by Jan Kara, Goldstein said; it would not require a sequence number because there is no need to notify user space that the change event has completed. Instead, the `FAN_PRE_MODIFY` is sent and the filesystem changes are made using sleepable read-copy-update (RCU); when user space wants to read, it makes a system call (perhaps called `vfs_write_barrier()`), which waits for all of the changes that were notified to complete. User space can then consume the batch of changes that have been made, without blocking any new changes from happening. 

David Howells asked what would happen if one of the changes failed after it had been notified. Goldstein said that false notifications are not a problem, as the server does not ""need to get the truth"". The change journal is not a journal that gets replayed, it simply records that something _may_ have changed in the directory. It is not recording any data, just the directory where changes may have happened. In his application, there are millions of files, so scanning for changes is not practical; ""you need to not miss changes and you need to not have races"". 

James Bottomley asked what was consuming the change journal. It can be used in various ways, Goldstein said, and there are filesystems that have the feature already. His application uses the journal after a reboot to ensure that the state of the filesystem is known; the journal is also periodically cleared once its entries have been processed. The pre-modification event is needed to ensure that changes get into the journal before the filesystem is modified, he said. 

He believes there is a need for the feature beyond his use case. There are likely others doing similar things, he said, using unreliable or probabilistic ways to detect that changes have occurred. He noted that the file change time (ctime) is modified in a way that is [problematic for NFS](<https://lwn.net/Articles/975863/>); it is done before the change is made on disk, so the client may observe the change before it has actually occurred, which means it can get out of sync. Layton agreed, suggesting that it would make sense to bump the ctime before and after a change, since the second operation will generally be a no-op. 

Concerns about performance were raised by several attendees. Goldstein said that there is some overhead, but that it is not large. If the feature is not of general interest, though, he can keep maintaining it in his tree. He does think there are other use cases for it, however; right now, filesystem indexing is relying on luck to a certain extent. He could not find another mechanism in Linux to ensure that the list of changes is consistent, which is why he has been working on it. He asked for preferences between the two options, a second event for change completion or a write barrier using sleepable RCU, but got [crickets](<https://idioms.thefreedictionary.com/crickets>) in response. 

An attendee asked about network filesystems and whether the notifications would work in that environment. There is a general problem with any kind of file notifications (inotify, fanotify) on the client side, Goldstein said, which needs to be solved at some point. On the server side, the usual mechanisms can be used, and the change-journal notification could also be used if it gets merged, but there is no way to communicate notification information to a client.
