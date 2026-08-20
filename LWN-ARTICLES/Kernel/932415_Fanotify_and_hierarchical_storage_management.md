---
title: Fanotify and hierarchical storage management
url: https://lwn.net/Articles/932415/
date: "May 23, 2023"
category: Storage management; fanotify
author: "By Jake Edge May 23, 2023 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 23, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

In the filesystem track of the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Amir Goldstein led a session on using [fanotify](<https://www.kernel.org/doc/html/latest/admin-guide/filesystem-monitoring.html>) for [hierarchical storage management](<https://en.wikipedia.org/wiki/Hierarchical_storage_management>) (HSM). Linux had some support for HSM in the XFS filesystem's implementation of the [data management API](<https://en.wikipedia.org/wiki/DMAPI>) (DMAPI), but that code was [removed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=288699fecaffa1ef8f75f92020cbb593a772e487>) back in 2010\. Goldstein has done some work on using fanotify for HSM features, but he has run into some problems with deadlocks that he wanted to discuss with attendees. 

He began by pointing to a [wiki page](<https://github.com/amir73il/fsnotify-utils/wiki/Hierarchical-Storage-Management-API>) he created to describe HSM and his goals for using fanotify to support it. His employer is CTERA Networks, which builds "cloud gateway solutions", where files appear to be available on the local system even though they may be cached on a local network-attached storage (NAS) device or stored somewhere else in the cloud. The NAS might not have space to accommodate all of the data, but it functions as a (more) local cache. 

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2023/lsfmb-goldstein-sm.png) ](<https://lwn.net/Articles/932649/>)

Windows has an API for HSM, so files have a status that reflects their location; users can decide if they want to access a file if, for example, it will require a lengthy copy from the cloud. This HSM support is based on "[reparse points](<https://learn.microsoft.com/en-us/windows/win32/fileio/reparse-points>)" in NTFS; when those are encountered, another filesystem driver is called to provide the file data. There is nothing like that in Linux, so those who provide that functionality have to implement their own scheme; CTERA uses [FUSE](<https://www.kernel.org/doc/html/next/filesystems/fuse.html>). 

The FUSE solution comes with various kinds of problems and he hopes that some of the alternatives being discussed at LSFMM+BPF will help alleviate them. DMAPI is an old API, which is insufficient for today's HSM needs, though the code from XFS still exists if there is anything useful in it; remnants of it are still present in Linux, as the "punch hole" interface was added for DMAPI. When the DMAPI hooks were removed, there was a comment suggesting that "at least the namespace events can be done much saner in the VFS", which is what Goldstein is trying to do now. 

He showed that a simple HSM can be implemented using the existing upstream fanotify API. It could use sparse files to represent the data that is not local. It does so by first getting an exclusive lock on the file object using [`fcntl(fd, F_SETLEASE, F_WRLCK)`](<https://man7.org/linux/man-pages/man2/fcntl.2.html>), migrating the content elsewhere, and then punching a hole in the file using `FALLOC_FL_PUNCH_HOLE` to [`fallocate()`](<https://man7.org/linux/man-pages/man2/fallocate.2.html>). The HSM service can subscribe to various types of fanotify events in order to be notified when the content, permissions, or directory entry of the file is changed; the cloud version can then be updated as needed. "It is very naive, but it works." 

However, it is not practical for today's use. For example, users have to download their entire movie, say, before starting to watch it. He has a patch set to add some features to fanotify that would make it more usable as an HSM, which he posted (as a pointer to his Git tree) in an [email](<https://lwn.net/ml/linux-fsdevel/CAOQ4uxhrQ7hySTyHM0Atq=uzbNdHyGV5wfadJarhAu1jDFOUTg@mail.gmail.com/>) back in September 2022\. The resulting thread eventually led to the session at the summit. 

The changes are small, he said, simply adding a few more fanotify event types (or additional information to existing events), which would facilitate the HSM use case. They are described further in a [section of the wiki page](<https://github.com/amir73il/fsnotify-utils/wiki/Hierarchical-Storage-Management-API#new-fanotify-events-and-information>) and would allow features like populating directories on demand, streaming downloads of large files, and crash-safe change tracking. He has been working on change tracking for a number of years now in various guises; he has an internal solution, but would like to get something into the mainline. 

He described a demo that he did not have time to actually perform, which can be seen in slide 6 of his [slides](<https://docs.google.com/presentation/d/1owJpX1rlXyXGchp3ioq-36VXcILDDoDZGpiYjm8ADEc/edit?usp=sharing>); it was based on the [HTTPDirFS](<https://github.com/fangfufu/httpdirfs>) FUSE filesystem, which allows read-only mounts of a directory accessed via HTTP. Goldstein modified it to use fanotify on a kernel with his patches. It would allow him to mount the kernel.org `/pub` directory locally, then access a file deep in the directory hierarchy. The filesystem lazily populates the needed directories into the local directory where it is mounted. The mount point is no longer a FUSE mount in that mode, but is a bind mount instead, with fanotify events being monitored. He displayed an example command that would display the first few lines of a `tar` table of contents of a large file. Only the first 1MB of the file would be transferred before the command completed, rather than waiting for the entire contents. 

He had two more slides after the "demo" slide, which were increasingly complex, he said. They were an attempt to explain some problems that he has found, "in order to try to sell the solution". At one time, there was a problem with the original fanotify API where an operation caused a `FAN_ACCESS_PERM` event, which might require the fanotify service to access the file; that results in a second (blocking) `FAN_ACCESS_PERM` event which leads to a user-space deadlock. That was solved by adding a special file descriptor that can be used by the service to perform actions without triggering another fanotify event. 

But now there is another deadlock that can happen with the existing API; it is perhaps rare, but it can happen and he is surprised that it has not been reported. It involves a [clone file range operation](<https://man7.org/linux/man-pages/man2/ioctl_ficlonerange.2.html>), which [takes the superblock freeze lock](<https://archive.kernel.org/oldlinux/htmldocs/filesystems/API-freeze-super.html>), but it may cause the HSM (or other fanotify-based) service to also need to freeze-lock the superblock. If the files are on the same filesystem (thus share the same superblock), a deadlock will result. 

This deadlock is perhaps more common in his HSM service than in other types of fanotify-based scanners (e.g. virus scanners). He has solved it by using a new event flag (`FAN_PRE_VFS`) that gets added to `FAN_ACCESS_PERM` events if the freeze lock has not been taken. He then went through and added that flag in the places where it was true, which involved calling the notify hook in some new places. That gives the service an opportunity to fill the file before the clone file range operation freezes the superblock. That was his solution, which was not hard to do, Goldstein said. 

He moved on to the second even-more-complicated slide, which covered a similar kind of deadlock, but it could also result in a race condition that would cause his HSM to miss filesystem changes at times. The scenario was well beyond my ability to follow it, but a video of the session should be available before long. His solution to the problem, which was suggested by Jan Kara, was to use [sleepable RCU](<https://lwn.net/Articles/202847/>), which would avoid the race at the cost of an occasional false-positive change notification. 

Once attendees seemed to get up to speed on the problem (and proposed solution), the session ran out of time, though discussion spilled over into the next slot. Josef Bacik said that he did not hate the solution that had been chosen, though he did not love it either. Kara explained why sleepable RCU was chosen, and Goldstein thought that the general idea could be applied to other filesystem-related ordering problems (such as when an inode's `i_version` field gets incremented).
