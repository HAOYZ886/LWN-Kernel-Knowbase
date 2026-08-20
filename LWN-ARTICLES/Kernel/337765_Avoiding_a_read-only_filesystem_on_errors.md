---
title: "Avoiding a read-only filesystem on errors"
url: https://lwn.net/Articles/337765/
date: "June 17, 2009"
category: Filesystems
author: "June 17, 2009 This article was contributed by Goldwyn Rodrigues"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

June 17, 2009

This article was contributed by Goldwyn Rodrigues

Errors in the storage subsystem can happen due to a variety of known or unknown reasons. A filesystem turns read-only when it encounters errors in the storage subsystem, or a code path which the filesystem code base should not have taken (i.e. a `BUG()` path). Making the filesystem read-only is a safeguard feature that filesystems implement to avoid further damage because of the errors encountered. Filesystems that change to read-only end up stalling services relying on data writes and, in some cases, may lead to an unresponsive system. In embedded devices, dying applications due to a filesystem turning read-only may render the device useless, leaving the user confused. 

Filesystem errors can either be recoverable or non-recoverable. Errors from bad block links in the inode data structures or block pointers can be recovered by filesystem checks. On the other hand, errors due to memory corruption, or a programming error, might not be recoverable because one cannot be sure which data is correct. 

Denis Karpov proposed a [patchset](<http://lwn.net/Articles/337492/>) that would notify user space, so that a user-space policy can be defined to take the appropriate action to avoid turning the filesystem read-only. The patchset is currently limited to FAT filesystems. User-space notifications would allow a filesystem to continue to be used after encountering "correctable errors" if proactive measures are taken to correct them. Such corrective actions can obviate the need for lengthy filesystem checks when the device is mounted next. 

The patchset adds a `/sys/fs/fat/<volume>/fs_fault` file which is initialized to 0 when the filesystem is mounted. The value of `fs_fault` changes to 1 on error. A user-space program can `poll()` on this file to check if the value of the file changed which is an indication that an error has occurred. Besides the file polling, a `KOBJ_CHANGE` uevent is generated, with the uevents environment variable `FS_UNCLEAN` set to 0 or 1. A udev rule can then trigger the correct program to take care of the damage done by the error. 

Kay Sievers is [not convinced](<https://lwn.net/Articles/337767/>) with the idea of using uevents for user-space notifications, as uevents are designed for device notifications, so they do not fit the design goals of reporting filesystem errors. Filesystem errors are quite often a repeated series of events. For example, a read failure may result in printing multiple read errors in dmesg for each block it is not able to read. An event generated for each block may be too much for udev to handle. Some of the events may get lost, or worse still, may cause udev to ignore uevents from other devices which occurred during the burst of errors. 

Uevents have no state, and the information is lost after the event. Uevents can not block, they need to finish in userspace immediately, you can not queue them up or anything else, it would block other operations. Uevents can _never_ be used to transport high frequent event streams. They might render the entire system unusable, if you have lots of devices and many errors. 

They could be used to get attention when a superblock does a one-time transition from "clean" to "error", everything else would just get us into serious trouble later. 

Keeping `<volume>/fs_fault` in sysfs is also not the best solution, because sysfs is unaware of filesystem namespaces. The primary responsibility of sysfs is to expose core kernel objects. Filesystem namespaces are a set of filesystem mounts that are only visible to a particular process and may be invisible to the rest of the processes. 

A process with a private namespace contains a copy of the namespace instead of a shared namespace. When the system starts, it contains one global namespace which is shared among all processes. Mounts and unmounts in a shared namespace are seen by all the processes in the namespace. A process creates a private namespace if it was created by the `clone()` system call using the `CLONE_NEWNS` flag (clone New NameSpace). A process sharing a public namespace can also create a private namespace by calling `unshare()` with `CLONE_NEWNS` flag. Mounts and unmounts within a private namespace are only seen by the process that created the namespace. A child process created by `fork()` shares its parent's namespace. 

Because of this, processes might receive errors on a filesystem in a different namespace, so they would not know which device to act on. The problem is also noticeable with processes accessing bind mounts created in a different namespace (bind mounts are a feature in which a sub-tree of a filesystem can be mounted on another directory). Moreover, filesystems spanning multiple devices, such as btrfs, would not be able to report the device name under the current naming structure. 

Kay [recommends](<https://lwn.net/Articles/337768/>) `/proc/self/mountinfo` as a better alternative, because it contains the list of mount points in the namespace of the process with the specified PID (self). Currently, `/proc/self/mountinfo` changes when the mount tree changes. This can be extended to propagate errors to user space in the correct namespace using `poll()`, regardless of the device name. `/proc/self/mountinfo` can accommodate optional fields which hold values in the form of "tag[:value]" that can be used to communicate the nature of the problem. Instead of using the existing udev infrastructure, this would require a dedicated application to monitor `/proc/self/mountinfo`, identify the error by parsing the argument, and act accordingly. 

Jan Kara further [suggests](<https://lwn.net/Articles/337769/>) using `/proc/self/mountinfo` as a link to identify the filesystem device generating the errors: 

What currently seems as the cleanest solution to me, is to add some "filesystem identifier" field to /proc/self/mountinfo (which could be UUID, superblock pointer or whatever) and pass this along with the error message to user-space. Passing could be done either via sysfs (but I agree it isn't the best fit because a filesystem need not be bound to a device) or just via generic netlink (which has the disadvantage that you cannot use the udev framework everyone knows)... 

Despite these objections, everyone agrees that error reporting to user space must not be limited to dmesg messages. User space should be notified of the errors reported by the filesystem, so that it can proactively handle errors and try to correct them. The namespace-unaware `/sys` filesystem or notifications through uevent may not be the best solution, but, for a lack of a better alternative interface, the developers used sysfs and uevents. The design is still under discussion, and will take some time to evolve, though it is likely that some kind of solution to this problem will make its way into the kernel.
