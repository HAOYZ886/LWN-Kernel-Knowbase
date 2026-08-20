---
title: Limiting negative dentries
url: https://lwn.net/Articles/1079407/
date: "July 3, 2026"
category: Dentry cache
author: "By Jake Edge July 3, 2026 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
July 3, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

A number of problems related to negative directory entries (dentries) were the topic of a filesystem-track session at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). Negative dentries are used to indicate that a file of a given name does not exist in a directory; it is an optimization that short-circuits the lookup of the file name when the answer is already known. Miklos Szeredi led a session that discussed some problems that come from having too many negative dentries for a directory. 

He began by noting that Ian Kent had reported a problem with hundreds of millions of negative dentries for a directory; in that case, the [`fsnotify_set_children_dentry_flags()`](<https://elixir.bootlin.com/linux/v7.1.2/source/fs/notify/fsnotify.c#L62>) call was made, which will iterate over all of the dentries in the directory and cause a soft lockup. A related issue is that the reference count field of the [`d_lockref`](<https://elixir.bootlin.com/linux/v7.1.2/source/include/linux/dcache.h#L114>) lock in [`struct dentry`](<https://elixir.bootlin.com/linux/v7.1.2/source/include/linux/dcache.h#L93>) can overflow if enough dentries are created, Szeredi said. It is not negative-dentry-specific, but it would be hard to create two-billion positive dentries for a directory. Kent had also mentioned that the hash chains may grow too long when there are so many negative dentries, but Szeredi is not sure that is a real problem. 

Amir Goldstein had previously suggested moving the negative dentries to the end of the [`d_children`](<https://elixir.bootlin.com/linux/v7.1.2/source/include/linux/dcache.h#L124>) list in `struct dentry`, Szeredi said. That would allow iterators like `fsnotify_set_children_dentry_flags()` to stop when they reach the first negative dentry. Chuck Lever was concerned that the order of child dentries might be exposed via [`getdents()`](<https://man7.org/linux/man-pages/man2/getdents.2.html>), thus moving the negative dentries might cause reordering in a way that would break users of `getdents()`; Szeredi did not think that would be a problem, however. An alternative might be to add calls to [`cond_resched()`](<https://elixir.bootlin.com/linux/v7.1.2/source/include/linux/sched.h#L2143>) into the code that walks the `d_children` list, Szeredi said; that would be more complex but would also address any soft lockups that are caused by positive dentries, not just those from negative dentries. 

Jan Kara was also concerned that moving negative dentries to the end might cause ordering problems if one of those dentries is changed to a positive dentry when a file is created. Some of the child-walking iterators depend on not missing any entries, he believes, so there could be problems inserting the updated dentry into the `d_children` list. ""I am not saying it's impossible, I am just not sure whether there will not be some catch."" Szeredi said that he thought the iterators' locking would prevent those kinds of problems as long as the dentry update also used that lock. 

David Howells raised the idea of switching the data structure used to track the children to something more suited to handling massive numbers of negative dentries. He did not have a concrete suggestion for what that data structure might be, however. Christian Brauner said that ""in principle there is nothing stopping us from changing the data structure"", though it would require agreement from others. 

Brauner asked about a [patch set from Kent](<https://lwn.net/ml/all/20260331012925.74840-1-raven@themaw.net/>) that would limit the number of dentries that could accumulate for the children of a directory. Brauner thinks that some kind of heuristic limit on dentries should be explored, separate from possible limits on negative dentries. That could perhaps lead to a ""meaningful heuristic that isn't just a sysfs file"" to better manage negative dentries. The problem with the accumulation of too many negative dentries has been around for ten years or more, he said; it is time to fix it. 

There is an opportunity to add to the existing [sysfs knob (`dentry-negative`)](<https://www.kernel.org/doc/html/latest/admin-guide/sysctl/fs.html#dentry-negative>), Brauner said. The current default is ""we will swamp your RAM with all of the dentries that we want"", which is fine for some workloads. Other users have complained about accumulating negative dentries, which is why `dentry-negative` can be set to one to cause [`unlink()`](<https://man7.org/linux/man-pages/man2/unlink.2.html>) to remove the dentry rather than turn it into a negative one. That does not solve the negative-dentry problem for everyone, however, so more options may be needed. 

Szeredi pointed out that user space would have to be in charge of managing that. ""Yes, that's my solution for everything"", Brauner said to laughter. It is often the case that user space is better placed to determine these kinds of things, he continued. ""Pushing everything out to user space, a la BPF, isn't a great solution always, but for a lot of that stuff we see that it actually works."" He pointed to the out-of-memory (OOM) killer as an example where the kernel really cannot make a sensible choice for all workloads so user space is better placed to do so. 

Jeff Layton agreed that policy choices should be left to user space. However, Ted Ts'o noted that doing so is in tension with ""decades of experience that users never change the defaults""; he suggested that sophisticated users have a knob available, but that the default work well for the common case. He thought that perhaps a limit of 1,000 negative dentries per directory would be sufficient for ""all sane workloads"", but surely ""my imagination is failing me"". He would like to find out about workloads where that limit would be a problem in order to understand what other options might make sense. 

The overflow of the reference count for the per-dentry lock, which came up as part of the too-many-dentries problem, is serious, Brauner said. It requires two-billion dentries, Kara said, so it is hard to hit. Brauner wondered if there could be some kind of built-in limit so that it could not happen. Layton asked if the count was actually overflowing, but Szeredi said it was a theoretical problem, not one that has been reported. 

Goldstein asked whether the superblock shrinkers prioritized negative dentries for eviction and reclaiming. Layton said that they probably do not since the dentries are maintained on a least-recently used (LRU) basis. Goldstein wondered if that should change. While Kara thought that perhaps it should, Szeredi pointed out that the problems occur on systems with too much memory so that the shrinkers are not being run at all. 

There was some unfocused talk about ways to limit the count and avoid the overflow as time ran out on the session.
