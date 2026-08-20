---
title: Trading off safety and performance in the kernel
url: https://lwn.net/Articles/643892/
date: "May 12, 2015"
category: Filesystems; O NOMTIME
author: "By Jonathan Corbet May 12, 2015"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 12, 2015

The kernel community ordinarily tries to avoid letting users get into a position where the integrity of their data might be compromised. There are exceptions, though; consider, for example, the ability to explicitly flush important data to disk (or more importantly, to avoid flushing at any given time). Buffering I/O in this manner can significantly improve disk write I/O throughput, but if application developers are careless, the result can be data loss should the system go down at an inopportune time. Recently there have been a couple of proposed performance-oriented changes that have tested the community's willingness to let users put themselves into danger. 

#### O_NOMTIME

A file's "mtime" tracks the last modification time of the file's contents; it is typically updated when the file is written to. Zach Brown recently posted [a patch](<https://lwn.net/Articles/643730/>) creating a new `open()` flag called `O_NOMTIME`; if that flag is present, the filesystem will not update mtime when the file is changed. This change is wanted by the developers of the [Ceph filesystem](<https://lwn.net/Articles/258516/>), which has no use for mtime updates: 

The ceph servers don't use mtime at all. They're using the local file system as a backing store and any backups would be driven by their upper level ceph metadata. For ceph, slow IO from mtime updates in the file system is as daft as if we had block devices slowing down IO for per-block write timestamps that file systems never use. 

Disabling mtime updates, Zach said, can reduce total I/O associated with a write operation by a factor of two or more. 

There are, of course, a couple of problems with turning off mtime updates. Trond Myklebust [noted](<https://lwn.net/Articles/643920/>) that it would break NFS ""pretty catastrophically"" to not maintain that information; NFS clients would lose the ability to detect when they have stale cached data, leading to potential data corruption. The biggest concern, though, appears to be the effect on filesystem backups; if a file's mtime is not updated when the file is modified, that file will not be picked up in an incremental backup (assuming the backup scheme uses mtime, which most do). A system's administrator might decide to run that risk, but there is the possibility that users may run it for them. As Dave Chinner [put it](<https://lwn.net/Articles/643921/>): 

The last thing an admin wants when doing disaster recovery is to find out that the app started using O_NOMTIME as a result of the upgrade they did 6 months ago. Hence the last 6 months of production data isn't in the backups despite the backup procedure having been extensively tested and verified when it was first put in place. 

Another way of putting it is that the mtime value is often not there for the benefit of the creator of the file; it is often used by others as part of the management of the system. Allowing the creator to disable mtime updates may have implications for those others, who would then have cause to wish that they had been part of that decision before it was made. 

Despite the concerns, most developers appear to recognize that there is a real use case for being able to turn off mtime updates. So the discussion shifted quickly to how this capability could be provided without creating unpleasant surprises for system administrators. There appear to be two approaches toward achieving that goal. 

The first of those is to not allow applications to disable mtime updates unless the system administrator has agreed to it. That agreement is most likely to take the form of a special mount option; unless a specific filesystem has been mounted with the "`allow_nomtime`" option, attempts to disable mtime updates on that filesystem will be denied. The second aspect is to hide the option in a place where it does not look like part of the generic POSIX API. In practice, that means that, rather than being a flag for the `open()` system call, `O_NOMTIME` will probably become a mode that is enabled with an `ioctl()` call. 

#### Syncing and suspending

Putting a system into the suspended state is a complicated task with a number of steps; in current kernels, one of those steps is to call `sys_sync()` to flush all dirty file pages back out to persistent storage. It might seem intuitively obvious that saving the contents of files before suspending is a good thing to do, but that has not stopped Len Brown from posting [a patch](<https://lwn.net/Articles/643926/>) to remove the `sys_sync()` call from the suspend path. 

Len's contention is that flushing disks can be an expensive operation (it can take multiple seconds) and that this cost should not necessarily be paid every time the system is suspended. Doing the sync unconditionally in the kernel, in other words, is a policy decision that may not match what all users want. Anybody who wants file data to be flushed is free to run `sync` before suspending the system, so removing the call just increases the flexibility of the system. 

This change concerns some; Alan Cox was quick to [point out](<https://lwn.net/Articles/643928/>) some reasons why it makes sense to flush out file data, including the facts that resume doesn't always work and that users will sometimes disconnect drives from a suspended system. It has also been pointed out that, sometimes, a suspended system will never resume due to running out of battery or the kernel being upgraded. For cases like this, it was argued, removing the `sys_sync()` call is just asking for data to be lost. 

Nobody, of course, is trying to make the kernel more likely to lose data. The driving force here is something different: the meaning of "suspending" a system is changing. A user who suspends a laptop by closing the lid prior to tossing it into a backpack almost certainly wants all data written to disk first. But when a system is using suspend as a power-management mechanism, the case is not quite so clear. If a system is able to suspend itself between every keystroke — as some systems are — it may not make sense to do a bunch of disk I/O every time. That may be doubly true on small mobile devices where the power requirements are strict and the I/O devices are slow. On such systems, it may well make sense to suspend the system without flushing I/O to persistent storage first. 

The end result is that most ([but not all](<https://lwn.net/Articles/643931/>)) developers seem to agree that there is value in being able to suspend the system without syncing the disks first. There is rather less consensus, though, on whether that should be the kernel's default behavior. If this change goes in, it is likely to be controlled by a sysctl knob, and the default value of that knob will probably be to continue to sync files as is done in current kernels.
