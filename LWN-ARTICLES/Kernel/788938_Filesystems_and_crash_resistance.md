---
title: Filesystems and crash resistance
url: https://lwn.net/Articles/788938/
date: "May 21, 2019"
category: Filesystems
author: "By Jake Edge May 21, 2019 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 21, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The "guarantees" that existing filesystems make with regard to persistence in the face of a system crash was the subject of a session led by Amir Goldstein at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). The problem is that filesystem developers are not willing to make much in the way of guarantees unless applications call [`fsync()`](<http://man7.org/linux/man-pages/man2/fsync.2.html>)—something that is not popular with application developers, who want a cheaper option. 

Currently, there are applications that create and populate a temporary file, set the attributes desired, then rename it, Goldstein said. The developers think that the file is written persistently to the filesystem, which is not true, but mostly works. The official answer is that you must use `fsync()`, but it is a "terrible answer" because it has impacts all over the system. 

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2019/lsf-goldstein-2-sm.jpg) ](<https://lwn.net/Articles/788940/>)

He wondered if there could be an API that gives the application developers what they are looking for. The [CrashMonkey](<https://github.com/utsaslab/crashmonkey>) developers did a bunch of testing on filesystem behavior after crashes, then brought some of the problems they found to the Btrfs developers, who said that they were not bugs. So the CrashMonkey folks wanted to [document the expected behavior](<https://lwn.net/ml/linux-fsdevel/1552418820-18102-1-git-send-email-jaya@cs.utexas.edu/>), then test and file bugs for filesystems that did not conform; that didn't work either, he said. He said it resulted in a long discussion between Dave Chinner, Ted Ts'o, and the Btrfs developers about the expected behavior, but there was a concern about committing to the existing behavior. 

So, perhaps there is a middle ground, Goldstein said. The kernel could add a new API via an `ioctl()` command or perhaps called `rename_barrier()` that provides the guarantees of the behavior that filesystems already provide today. Later, if there is an optimization that changes the existing behavior, simply do not add that optimization to the path that the new API uses. 

There are two types of barriers that he is talking about. The first would be used by overlayfs; it sets extended attributes (xattrs) on a file, then renames it. Overlayfs expects that, if the rename is observed, then the metadata has been written persistently. The other barrier is for data to be persistently written, which can be done today using the [`FIEMAP` `ioctl()` command](<https://www.kernel.org/doc/Documentation/filesystems/fiemap.txt>) (with the `FIEMAP_FLAG_SYNC` flag), at least for XFS and ext4, he asserted. 

But Chris Mason said that won't work for sparse files even on ext4 and XFS. Jan Kara said that it is a side effect of how ext4 and XFS do their journaling; the data will get to disk and the metadata will go into the journal. It is cheaper than an `fsync()`, he said. 

Ric Wheeler is concerned that filesystem developers have "spent decades" trying to explain how to use `fsync()` to application developers without success, so any new mechanism will simply be "so mysterious" that no one will use it. But Goldstein disagreed, saying that the mechanism is "totally natural". Wheeler, however, was not convinced; if he was to put things in a "more mean" way, he would say that the application developers do not understand what they expect. 

Kara said that he agreed with Wheeler that a new API is not going to solve the problem. Even if it were all documented, Ts'o is skeptical that application authors will read and internalize the documentation; "we have ample evidence" that they do not. But Goldstein said that he is an application developer and he needs an API for this. Right now he is using `FIEMAP` with the sync flag, but that is not an API, he said; `FIEMAP` was added for an entirely different purpose and the flag was only meant to ensure the extents had been allocated. Mason said that he "would love it" for rename to work the way the application developers want it to, but it is too expensive. Adding a new way to accomplish what he thinks the applications want, but costs more than a rename, would be fine with him. 

Ts'o said that kernel developers need to carefully document whatever they are going to do; even then "we are still going to lose". He doesn't want to repeat the experience of [`sync_file_range()`](<http://man7.org/linux/man-pages/man2/sync_file_range.2.html>), which didn't do exactly what people expected it to do, but applications ended up depending on things it didn't actually do. Two new operations, `fbarrier()` and `fdatabarrier()`, have been proposed in several academic papers lately, but the semantics differ among the papers. Before committing to a name and the behavior, it may be worthwhile to look at these papers, he said. 

Goldstein said that the CrashMonkey developers have already documented some of what is needed, but Ts'o said that he hated that document; "that is not a starting point", he said. It is based on what the developers observed, rather than on a specification of a new system call and what guarantees it provides—and does not provide. Observing what ext4, XFS, and Btrfs do today and expecting that to continue into the future "is not the way to do this". 

The idea is just to have an API that guarantees what ext4 already does today, Goldstein said; there should be no implementation cost. But Ts'o cautioned that should not be how it is framed. Instead, a specification should be written first, then debated, before an implementation is done. He is not in favor of the feature, but if people want to proceed, this is the way to "minimize the blast radius". Ts'o believes the feature will be "misused more than it is used". 

Wheeler is worried about making a new API that will be hard to explain; right now, there are ways to accomplish the same thing without the API. But Goldstein said it is a weird situation to be hiding this kind of thing from users. Ewan Milne said that presenting a new API that will be seen as a "faster `fsync()`" that doesn't do everything that `fsync()` does will just serve to confuse. 

The problem is getting worse, Goldstein said, as `fsync()` is getting more expensive. Since there are already ways to get the behavior that he and other application developers want, without changing anything, it makes sense to expose it. There are advanced users of the filesystems (e.g. overlayfs, PostgreSQL, Git) that will benefit from it. 

In the end, the outcome of the discussion seemed rather inconclusive. Later in the conference, though, Goldstein made a point of letting me know that some discussions with Ts'o and others after the session did come to some level of agreement on the path forward.
