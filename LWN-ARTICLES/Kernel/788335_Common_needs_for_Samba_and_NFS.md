---
title: Common needs for Samba and NFS
url: https://lwn.net/Articles/788335/
date: "May 15, 2019"
category: "Filesystems-NFS; Samba"
author: "By Jake Edge May 15, 2019 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 15, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Amir Goldstein led a discussion on things that the two major network filesystems for Linux, Samba and NFS, could cooperate on at the end of day one of the 2019 Linux Storage, Filesystem, and Memory-Management Summit. In particular, are there needs that both filesystems have that the kernel is not currently providing? He had some ideas of areas that might be tackled, but was looking for feedback from the assembled filesystem developers. 

He has recently just started looking at the kernel NFS daemon (knfsd) as it is a lesser use case for the customers of his company's NAS device. Most use Samba (i.e. SMB). He would like to see both interoperate better with other operating systems, though. 

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2019/lsf-goldstein-sm.jpg) ](<https://lwn.net/Articles/788339/>)

He noted that he had asked some Samba developers why it has been hard to get the features they need into the kernel. The "vibe" he got is that they were rather intimidated by the kernel community. But he believes that the last time they tried, "they did it wrong". They wanted Samba-specific features, such as preventing writes or reads to a specific file, to be enforced by the VFS. Those kinds of changes were not acceptable to the VFS maintainers, however. 

In talking with some NFS developers, it was agreed that Samba and NFS should "talk amongst themselves" to find areas where both needed kernel support. There are likely things that either can only be done in the kernel or are far better done in the kernel. If there is a minimal set of infrastructure that the kernel could provide to help solve those problems, it may be possible to get it added. 

He started with [opportunistic locks](<https://docs.microsoft.com/en-us/windows/desktop/fileio/opportunistic-locks>) (OpLocks). In SMB, these locks can be requested when the client takes a lease on a file. If granted, that means the client does not need to flush its changes to the server as long as it holds the lock. If another client is accessing the file, the server can revoke the OpLock. There are large performance gains that OpLocks provide, so it is important to be able to fully support them. 

Samba will use OpLocks, but only if it is configured in the mode that it will be the only user of the filesystem it is serving. If, say, an NFS server could also be touching the filesystem, Samba does not use OpLocks at all. There are also [two levels of OpLocks](<https://docs.microsoft.com/en-us/windows/desktop/fileio/types-of-opportunistic-locks>); Samba uses both when it is configured as the only thing touching the filesystem. 

Steve French said there is more to it than that. OpLocks are identified with a key, which would allow level-1 locks to be upgraded to level 2, but doing an upgrade is not implemented for Linux. He was not sure whether NFS could upgrade its locks or not. Trond Myklebust said that the NFS protocol allows upgrading, but that it is not implemented for Linux. 

So Goldstein wondered what Samba and NFS need in order to be able to implement level 2\. He plans to go to the Samba conference ([sambaXP](<https://sambaxp.org/>)) in June and will gather more information there. One thing he knows needs to be done is to be able to track clients through the filesystem operations in order to manage their leases, which is similar to what Bruce Fields described for NFS in an [earlier talk](<https://lwn.net/Articles/788292/>). That would take care of file leases. 

For directory leases, Goldstein has been working on a way to get notifications of directory changes. He has hooks for doing synchronous notifications, but there are sensible concerns about exporting those, he said. He thinks that perhaps exposing them in a way similar to leases, with a timeout in case there is no reply, might be possible. 

One of the problems that NFS has with leases, Myklebust said, is that it doesn't detect that they have timed out. If you miss your notification window, ideally the file descriptor would be closed or something else would "fence off" access to the file. That would be an indication that some kind of recovery action needs to be taken. Goldstein said that he is not tied to any specific solution; he is just bringing up areas that might need to be addressed. 

Next up was [share modes](<https://docs.microsoft.com/en-us/rest/api/storageservices/managing-file-locks>) that can be specified when a file is opened. Those modes can request that any other opens of the file, for say read, write, or both, will fail. There is a patch set from five years ago that implemented a flavor of mandatory lock to support these modes, but it took the approach of enforcing the modes in the VFS. 

Goldstein thinks that Samba and NFS could cooperate on a new flavor of lock that only they would use. It would be sort of similar to the BSD [`flock()`](<https://www.freebsd.org/cgi/man.cgi?query=flock&sektion=2&manpath=freebsd-release-ports>) system call, he said. New `open()` flags would be added to request this exclusive access (e.g. `O_DENYWRITE` and `O_DENYREAD`), but they would only be implemented for filesystems that want to enforce that. 

Ted Ts'o said that some kind of document outlining the various problems is going to be needed. He suggested that it would be helpful to explain what the downsides to not implementing those features would be. Samba works pretty well, so explaining why kernel developers should care about these additional features may help smooth their path. 

Goldstein said that he was mostly just trying to inform the other filesystem developers of his plans. Overall, he does not expect all that much to come of this effort, but "maybe we can move the needle". His list is not comprehensive by any means; he does not know if such a thing exists. Mainly, he wants to start the conversation about these network filesystem needs.
