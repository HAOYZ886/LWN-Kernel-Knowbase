---
title: "Hierarchical storage management, fanotify, FUSE, and more"
url: https://lwn.net/Articles/981392/
date: "July 16, 2024"
category: "Filesystems-In user space; Storage management; fanotify"
author: "By Jake Edge July 16, 2024 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
July 16, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Amir Goldstein led a filesystem-track session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) on his project to build a [hierarchical storage management](<https://en.wikipedia.org/wiki/Hierarchical_storage_management>) (HSM) system using [fanotify](<https://www.kernel.org/doc/html/latest/admin-guide/filesystem-monitoring.html>). The idea is to monitor file access in order to determine when to retrieve content from non-local storage (e.g. the cloud). The session was a follow-up to last year's [introduction to the project](<https://lwn.net/Articles/932415/>), which covered some of the problems he had encountered; this year, he was updating attendees on its status and progress, along with some other problem areas that he wanted to discuss. 

#### Background

He began with a quick overview of the use case for HSM and how his project fits in. A file that is stored in the cloud, say, can appear to be local to the system. When the file is accessed, an fanotify listener reacts to that event to make the file available locally. Once the data requested has been filled into the file, the listener can return to the program that accessed the file. 

Last year, he described a problem with a deadlock that can happen under some circumstances when the listener daemon starts writing data to the file after it has been notified that the file has been accessed. To fix that, he took some of the advice that had been offered and [moved the fsnotify hooks outside of the VFS locks](<https://lwn.net/ml/all/20240105-vfs-rw-9b5809292b57@brauner/>), which resolved that; those patches were merged for Linux 6.8. He has had those patches available for a while, and several developers had incorporated them into their testing, including the kernel-test robot; he was thankful for those efforts, in part because the robot found some performance regressions due to new hooks that were added. He was able to restructure the fsnotify code and get that into 6.9, which reduced the overhead, both for his use case and for other users. 

He has been working on some "pre-content events" for fanotify for a while; several developers have reviewed and tested that code, Goldstein said. The current code lives in his ["fan_pre_content" branch](<https://github.com/amir73il/linux/commits/fan_pre_content>) at GitHub. The new `FAN_PRE_ACCESS` and `FAN_PRE_MODIFY` events have some new `FAN_EVENT_INFO_TYPE_RANGE` information attached to them. It describes the range of the file that is affected. In his proof-of-concept (PoC) implementation that is also on GitHub, those events allow opening a large file that is located elsewhere on the net, such as a Linux tarball, and reading just the first part of it. 

His branch has also added the ability for fanotify to return errors other than `EPERM`. Currently, the listener can only return success or `EPERM`, but with his changes other errors, such as `EBUSY` and `ENOSPC`, can be returned. 

#### Open questions

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2024/lsfmb-goldstein-sm.png) ](<https://lwn.net/Articles/981857/>)

He uses real filesystems, such as Btrfs or XFS, as the local storage for his HSM; when the files are not populated, they are sparse files locally, thus use no storage space. If the HSM daemon has not been started, however, he does not want users to be able to access those files. One idea to prevent that is to have a "moderated mount" using an HSM mount option that would only allow access when the daemon is running; accesses at other times would result in `EPERM`. He wondered if attendees had thoughts or opinions on that. 

As part of his work on HSM, he looked at what other operating systems have done; Windows has long had HSM support that is based on NTFS "[reparse points](<https://learn.microsoft.com/en-us/windows/win32/fileio/reparse-points>)", which are persistent marks on files that refer to the driver needed to access them. Perhaps Linux could do something similar by adding some kind of persistent mark to files instead of using the moderated-mount idea. 

Josef Bacik said that he did not like the mount-option idea because options cannot be applied differently to separate subvolumes; containers can have access to different Btrfs subvolumes, which would mean they all need to be treated the same way. Goldstein suggested that containers could use different bind mounts for the subvolumes, with different options applied, which Bacik agreed would work. 

But Bacik prefers the persistent-mark idea, in part because of some problems he has encountered in executing files that get filled in via the fanotify technique. When using [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>) to execute a remote file, there is no opportunity for the HSM to write the data into the file; by the time the writes can be done, `execve()` has mapped the memory such that an `ETXTBUSY` error is returned for any writes. Both Bacik and Goldstein noted that Btrfs has an `ioctl()` command that can be used to avoid the problem, which is where Bacik sees the persistent-mark feature as a way around the problem. But that is a filesystem-specific solution and Goldstein would like to be able to solve the problem in a more general way. 

Goldstein said that he had two different versions of the PoC code in his tree (some of which is [described in his HSM wiki](<https://github.com/amir73il/fsnotify-utils/wiki/Hierarchical-Storage-Management-API>)); one uses evictable marks on the inode to mark files that have been fully populated, while the other implements a form of persistent marks using extended attributes (xattrs). There is already precedent for fanotify to use protected xattrs in the "security.fanotify" namespace, so he added a "mark" xattr there. 

Jan Kara was not in favor of the persistent marks as xattrs, however. He was concerned about who would own the mark; if there are multiple HSM implementations, how would the kernel decide which to use for a given file? For Windows, the reparse point includes an identifier to specify what storage driver needs to be used for the file, Goldstein said. That could work, Kara said, though it is a bit ugly for VFS code to have to root around in xattrs 

#### `execve()`

The conversation turned back to the `execve()` problem and Bacik said that he had another solution for that, which he suspected that many would hate even more. His employer, Meta, wants to be able to execute binaries that may not be present locally, so adding a persistent mark that would tell the system to invoke a [usermode helper](<https://kernelnewbies.org/KernelProjects/usermode-helper>) to populate the file would make that possible; it would not use fanotify at all, he said. With a chuckle, Bacik said that he was ignoring the screams from Christian Brauner in the background. Kara suggested a new system call, which Bacik thought might be "a more palatable solution". 

Brauner asked for more clarification of the problem being solved, which revolves around the restrictions on writable memory being mapped for execution. Something needs to write the contents of the file in order to populate it, but the memory to be used by `execve()` is in a private mapping, Aleksa Sarai said, so it cannot be populated later. A usermode helper could route around that, Bacik said. 

Brauner said that his main concern is that the usermode helper feature is not namespace-aware, which has security and other implications. Basing the API around that has a lot of impact on extensibility and maintainability in the future, which also concerns him. Bacik agreed, but said that the usermode helper is initiated from the kernel side so he thought it might be more acceptable rather than allowing the fanotify listener to do so from user space. Brauner suggested that looking at the usability and security angles should help determine which is the better approach. 

But the problem is deeper, Goldstein said. The file is marked as being "deny write" once it is opened by `execve()`, so nothing else can successfully even open the file. Getting around that, while still going through the VFS layer, would break some fundamental pieces of that layer, he said. Writing to the file by way of a Btrfs `ioctl()` command could work, as would a new system call that gets around the VFS, but populating the file using `write()` will not. 

#### FUSE

Moving on, Goldstein noted that FUSE can be used instead of fanotify and it is much more flexible, but it requires that all of the I/O flow through the FUSE daemon. That does not perform as well as people want, which is why he has been working with others on [FUSE passthrough](<https://lwn.net/Articles/932060/>). So FUSE could be another path toward an HSM solution. 

Bacik said that he was not sure which path he would be taking, but he would be investigating both. "I have money on both horses", Goldstein said to laughter. Ted Ts'o noted that Dan Rosenberg has been working on the [FUSE BPF filesystem](<https://lwn.net/Articles/937433/>), which is aimed at Android being able to do incremental loading of executables; "so there's actually some other work happening in this space". Goldstein said that FUSE passthrough had just been merged for Linux 6.9 and was partly based on the Android patches. The plan is for the BPF pieces to come in once the FUSE passthrough feature has stabilized. 

Bacik said that FUSE makes sense for Android, but does not for Meta, at least for some of its uses. If the FUSE daemon crashes, suddenly applications start crashing in production; the fanotify solution is more self-contained. Goldstein said that for "mostly passthrough filesystems", the fanotify solution is probably better—"if we can get it to work", he said with a chuckle as time expired. 

The [YouTube video](<https://www.youtube.com/watch?v=qY_e6Sev1vg>) of the session and Goldstein's [slides](<https://docs.google.com/presentation/d/1uacbNXA5UQGT6QxjjePDvHhqB89Wk6ybb2wCEfA5Ay0/>) are available.
