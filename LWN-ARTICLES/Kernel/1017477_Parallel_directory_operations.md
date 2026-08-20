---
title: Parallel directory operations
url: https://lwn.net/Articles/1017477/
date: "April 16, 2025"
category: "Filesystems-Virtual filesystem layer"
author: "By Jake Edge April 16, 2025 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
April 16, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Allowing directories to be modified in parallel was the topic of Jeff Layton's filesystem-track session at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF). There are certain use cases, including for the NFS and Lustre filesystems, as mentioned in a [patch set](<https://lwn.net/ml/all/20241220030830.272429-1-neilb%40suse.de/>) referenced in the [topic proposal](<https://lwn.net/ml/all/f78f4a5e86c10d723fd60d51a52dd727924fed3a.camel%40kernel.org/>), where contention in creating multiple files in a directory is causing noticeable performance problems. In some testing, Layton has found that the inode read-write semaphore (`i_rwsem`) for the directory is serializing operations; he wanted to discuss alternatives. 

He and Christian Brauner had worked on a [patch set](<https://lwn.net/ml/all/20240807-openfast-v3-1-040d132d2559%40kernel.org/>) that addressed some of the low-hanging fruit from the problem. It would avoid taking the exclusive lock on the directory if an open with the `O_CREAT` flag did not actually create a new file. It provided a substantial improvement for certain workloads, Layton said. 

But what is really desired, he said, is to be able to do all sorts of parallel directory operations (mkdir, unlink, rmdir, etc.). Neil Brown developed the patch set referenced earlier (which was [up to v7](<https://lwn.net/ml/all/20250206054504.2950516-1-neilb%40suse.de/>) as of the summit) that would stop taking the exclusive (write) lock for every operation on the directory; it would take the shared (read) lock instead. There are filesystems, such as NFS, that do not need to have those operations serialized, Layton said. 

Brauner and Al Viro reviewed Brown's patch set and gave some good feedback, Layton said. The stumbling block for those patches is that they take the directory `i_rwsem` as a shared lock, rather than as an exclusive one, but take an exclusive lock on the directory entry (dentry), which opens up a lot of possibilities for deadlock. There are lots of corner cases, so the problem is with making it ""provably correct, which is going to be an order of magnitude more difficult than the existing exclusive lock"". 

[ ![\[Jeff Layton\]](https://static.lwn.net/images/2025/lsfmb-layton-sm.png) ](<https://lwn.net/Articles/1017678/>)

Brown has been working on the patches for a while. The most recent version adds many new inode operations with `_async` added, ""which is a little ugly"", Layton said, but that is ""cosmetic stuff"". The hard part is getting the locking right. 

That is where things stand, he said, and wondered what filesystem developers could do to help with the ""long slog"", because it is important for scalability in directories. The workloads where the problems are seen are ""big untar-type workloads"", where there are many threads all creating files in parallel in the same directories. 

Brauner asked if individual filesystems would have to opt into supporting the feature. Layton said that he did not see any other possibility. One path might be to, say, create a `mkdir_async()` inode operation, slowly convert all of the filesystems to use that, and then replace the `mkdir()` operation. Brauner said that he did not like adding lots of new inode operations, but thought that if the idea was viable, the existing operations could add an asynchronous form that would only be used for filesystems that opt in. Layton suggested starting slow, with a single inode operation for unlink, say; Brauner thought that sounded like a reasonable approach. 

Josef Bacik wondered if it made sense to push the locking down to individual filesystems; ""I hate doing this"" but it would mean that filesystems could try alternative locking that would allow more parallelism. There would need to be code that embodies the existing locking for every filesystem (except those trying out alternatives) to use. 

Mark Harmstone asked about batching the operations for these workloads, which could be done, but requires rewriting applications. Brauner pointed out that io_uring could probably be used to do the batching today if that was desirable, but Harmstone said that he had been using io_uring for directory-creation (mkdir) operations which still takes locks for each one. Layton said that a fast-path option could perhaps be added to io_uring, but there is a more general problem to be solved. 

For example, if multiple files are created in the same directory using NFS, those operations are all serialized on the client side, which means that it requires lots of network round trips before anything can happen. Fixing that would still end up serializing many of the operations on the server side, but that would be better. Bacik concurred, noting that you can take the untar workload out of the picture; creating a file in a directory takes an exclusive lock, so any lookups that involve the directory are held off as well. 

Part of the problem is that it is not entirely clear what `i_rwsem` is protecting, Brauner said. For example, the setuid exit path recently started taking that lock to avoid some potential security problems. David Howells said that it is also used by network filesystems to ensure that direct and buffered I/O do not conflict. 

Chris Mason asked which local filesystems would be able to handle parallel creation in directories. Layton did not know of any, but Bacik said that Btrfs would be able to; he agreed when someone else suggested that bcachefs might as well. Timothy Day from the Lustre filesystem project said that it could do so as well. 

Bacik noted that once parallel creates are in place, ""for sure we are going to run into the next thing""; there will be other constraints that will be encountered. Bacik said that network filesystems are likely to be able to more fully take advantage of the feature, while local filesystems will only have modest gains. He has an ulterior motive in that [Filesystem in Userspace](<https://www.kernel.org/doc/html/latest/filesystems/fuse.html>) (FUSE) servers may be able to find better locking schemes in user space if the kernel is not taking exclusive locks on its behalf.
