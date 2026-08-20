---
title: The conclusion of the 3.5 merge window
url: https://lwn.net/Articles/500212/
date: "June 5, 2012"
category: "Releases-3.5"
author: "By Jonathan Corbet June 5, 2012"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 5, 2012

Linus closed the 3.5 merge window on June 2; the [announcement](<https://lwn.net/Articles/500025/>) for the 3.5 prepatch came one day later. Just under 1,000 patches were pulled into the mainline after the writing of [last week's merge window summary](<https://lwn.net/Articles/498693/>), for a total of 9,534 for the merge window as a whole. Some of the more significant changes pulled at the end of the merge window include: 

  * The HFS filesystem has gained native language support (NLS) capabilities, with codepages added for several languages. 

  * A process's directory under `/proc` now includes a `children` file containing the IDs of its child processes. 

  * The `kcmp()` system call has been added. Its purpose is to help user space checkpoint/restore utilities to determine whether two processes share a given resource or not; see [this article](<https://lwn.net/Articles/478111/>) for a description of the interface. 

  * Also for checkpoint/restore: the `prctl()` system call has gained options to set the beginning and end of the argv and environment areas and the executable file a process is running. 

  * The ext4 filesystem now has support for metadata checksumming, a feature which should help to catch metadata corruption that might otherwise escape notice. This feature requires an on-disk format change (to store the checksum), so it must be turned on explicitly. Once this feature is enabled, the filesystem can only be mounted read-only on older kernels. See [this article](<https://lwn.net/Articles/469805/>) for more information on the metadata checksum feature. 

  * A lot of changes have gone in to improve the handling of system-specific features on Sony laptops. 

  * The new `skew_tick=` boot option controls whether the system skews timer ticks on a per-CPU basis to minimize contention on the `xtime_lock` lock. It defaults to off; turning it on can reduce jitter on some workloads, but will also increase power consumption. 

  * The "frontswap" mechanism, part of the [transcendent memory](<https://lwn.net/Articles/454795/>) family of technologies, sneaked its way into the mainline just after the -rc1 release. 

  * The FUSE filesystem API has added an operation to implement the `fallocate()` system call. 

  * Two new drivers were added at the end of the merge window; they enable AUO K1900 and K1901 epaper display controllers and Emma Mobile STI timers. 

Changes visible to kernel developers include: 

  * The `task_work_add()` function, useful for requesting that a function be run in the context of a specific process, has been added. See [this article](<https://lwn.net/Articles/494158/>) for a description of the `task_work_add()` API. 

  * The SUNRPC code has a new utility function: 

```
int svc_bind(struct svc_serv *serv, struct net *net);
```

Its purpose is to handle service registration in the given network context; `svc_bind()` is exported GPL-only. 

  * `struct inode_operations` has a new `update_time()` function whose job is to provide any needed special handling for changes to any of the file timestamps. The `file_update_time()` prototype has been changed: it now returns an `int` that can indicate that the operation failed. Failures to update the last-access time are now explicitly ignored; this is done to ensure that atime update failures don't make the filesystem unreadable. This work has been generalized out of the btrfs filesystem; see [this article](<https://lwn.net/Articles/499293/>) for a description of how atime updates can go wrong otherwise. 

At this point, the addition of features for the 3.5 development cycle should be at an end unless something sneaks in before -rc2. If the usual pattern holds, the final 3.5 release can be expected right around the beginning of August.
