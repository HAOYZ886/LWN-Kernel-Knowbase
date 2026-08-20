---
title: Topics from the virtual filesystem layer
url: https://lwn.net/Articles/1017128/
date: "April 16, 2025"
category: "Filesystems-Virtual filesystem layer"
author: "By Jake Edge April 16, 2025 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
April 16, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

In the first filesystem-track session at the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF), virtual filesystem (VFS) layer co-maintainer Christian Brauner had a few different topics he wanted to talk about. Issues on the agenda included iterating through anonymous mount namespaces, a needed feature for ID-mapped mounts, the perennial unprivileged mounts topic, potentially using hazard pointers for file reference counting, and Rust bindings. He did not expect to get through all of them in the 30 minutes allotted, but the session did move along pretty quickly to at least introduce them to the assembled filesystem developers. 

He noted that one of the accomplishments for the filesystem community over the last few years was in [reworking the mount process and API](<https://lwn.net/Articles/979166/>). He was hopeful that [mount notifications](<https://lwn.net/Articles/980330/>) would be merged during the 6.15 merge window that was taking place during the summit—and they were. That feature will be useful for getting notifications of changes to mount trees, rather than having to frequently query the kernel or `/proc` files to keep track. 

#### Anonymous mount namespaces

[ ![\[Christian Brauner\]](https://static.lwn.net/images/2025/lsfmb-brauner-sm.png) ](<https://lwn.net/Articles/1017478/>)

In the VFS, there is a notion of anonymous mount namespaces (""or 'detached mount trees', however you want to look at it"") where a mount namespace exists, but processes cannot use [`setns()`](<https://www.man7.org/linux/man-pages/man2/setns.2.html>) to enter it; it is not attached to anything else. Oddly, though, a process can [`chroot()`](<https://man7.org/linux/man-pages/man2/chroot.2.html>) to a directory in it, but then cannot list the mounts in the namespace. These mount namespaces do not have a representation in `/proc/PID/mountinfo`, which makes them ""completely opaque to user space""; there is no way to interact with them, he said. That is a problem because, for example, a process can have a file descriptor for such a namespace, which pins various things into memory, but there is no way for user space to figure out what is responsible for pinning the memory. 

So there is a need to expand the [`listmount()`](<https://www.man7.org/linux/man-pages//man2/listmount.2.html>) and [`statmount()`](<https://www.man7.org/linux/man-pages//man2/statmount.2.html>) system calls to be able to interact with anonymous mount namespaces. There is now a [way to iterate through all mounts](<https://brauner.io/2024/12/16/list-all-mounts.html>) in all mount namespaces, except those attached to anonymous mount namespaces. Container workloads, under Kubernetes in particular, can have hundreds of mount namespaces; prior to the addition of `listmount()` and `statmount()`, listing all of the mounts would have required concatenating the output from all of the `/proc/PID/mountinfo` files. Extending that to the anonymous mount namespaces will help complete the API, he said. 

Jeff Layton asked if the anonymous mount namespaces are collected onto a list in the kernel somewhere, but Brauner said they are not currently. There is a red-black tree for the mount namespaces, but that is indexed by the sequence number assigned to the namespace; anonymous mount namespaces all have a sequence number of zero. Instead of using the zero to recognize anonymous entries, a flag or something should be used, then a regular sequence number can be assigned and the entries can be added to the tree, Brauner said. 

Layton said that he has been running into a similar problem with network namespaces; NFS can take a reference to a disconnected network namespace that then stays alive with no way to track down what has the reference. Brauner said that he had discussed this problem with Josef Bacik recently and they agreed that all namespaces should follow the plan for mount namespaces and assign a sequence number for each; the [`struct ns_common`](<https://elixir.bootlin.com/linux/v6.13.7/source/include/linux/ns_common.h#L9>) that holds namespaces would have the sequence number (or some kind of identifier) added to it, so that namespaces can be added to data structures and operated upon. It is a generic problem for namespaces that should be solved in a unified way, Brauner said. 

#### ID-mapped mounts

Another thing he has been working on is a follow-on to the [ID-mapped mounts](<https://lwn.net/Articles/896255/>) feature that was merged into the 5.12 kernel in 2021. In 6.15, the ability to [change UID and GID mappings for an ID-mapped filesystem](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7a54947e727b>) by performing another ID-mapped mount on it was added. Now there is a need for various squashing options, where, for example, a range of IDs all map to a single ID. 

One of the problems with the existing ID-mapping mechanism is that every ID present in the filesystem needs to be explicitly mapped to another ID or a file with an unmapped ID has no ID—it cannot be interacted with at all. For efficiency reasons, there may need to be limits on the number of squashed ranges that can be supported, but some kind of simple range squashing is needed. In addition, a way to say that any unlisted IDs map to a single specific ID should be supported. 

The interface for specifying mappings currently follows the model that [user namespace](<https://www.man7.org/linux/man-pages/man7/user_namespaces.7.html>) mappings use, where mapping information is written to `/proc` files. That works, but Brauner does not think it will scale to the needs of ID-mapped mounts. His immediate focus is to add a way to squash all of the unmapped IDs to a specified ID; he has a proof-of-concept implementation in his tree that passes all of his tests. It will make things much simpler for use cases like mapping all of the IDs on a filesystem to a single ID as no lookup will be needed. 

#### Unprivileged mounts

Brauner informally polled the audience for which topic the attendees wanted him to talk about next: unprivileged mounts or file reference counts. Several spoke up for the mount topic and any reference-count fans were notably silent. ""Dammit, I wanted to pitch hazard pointers"", he said with a laugh, though he did eventually get to the topic. 

There are two aspects to the unprivileged-mount problem; the first is to allow unprivileged users to mount ""a random USB stick"", which is ""a really terrible idea"". The other is to mark a specific filesystem as being mountable inside of a user namespace, ""which adds a bit more protection, at least in terms of setuid binaries and all that kind of stuff"", he said, but it does not help with the problems of malicious filesystem images. He does not think that there is any solution for allowing unprivileged users to mount random filesystem images; ""I don't believe that Rust will solve this problem, I think that's a pipe dream"". 

Brauner pointed to the solution described in an LSFMM+BPF [session from 2023](<https://lwn.net/Articles/934176/>), which allows user space to ""safely delegate mounting of filesystems to unprivileged users"" using [systemd-mountfsd](<https://www.freedesktop.org/software/systemd/man/latest/systemd-mountfsd.html>). That solution does not work for network filesystems, but Layton said that was simply a policy decision; the same mechanism could be used but network filesystems would need to add some capabilities to enable it. 

For the USB-stick case, Brauner said, the solution should be to use [Filesystem in Userspace](<https://www.kernel.org/doc/html/latest/filesystems/fuse.html>) (FUSE) and ""don't mount untrusted stuff"". Over the remote link, Jan Kara said that the solution for USB mounts does not lie in the kernel. OpenSUSE has started looking into [mounting USB sticks using the Linux Kernel Library (LKL)](<https://hackweek.opensuse.org/projects/usb-storage-plumbing-for-the-linux-kernel-library>), which is somewhat similar to [User Mode Linux](<https://www.kernel.org/doc/html/v5.9/virt/uml/user_mode_linux.html>); it has a FUSE daemon that uses LKL to mount the filesystem and expose it to the kernel, which ""provides additional isolation"". For USB sticks, performance is not particularly important, he said, so this ""seems like a promising solution"" 

#### Reference counts

Last year, Brauner said, he [added a reference-count mechanism](<https://lwn.net/ml/all/20241007-brauner-file-rcuref-v2-0-387e24dc9163@kernel.org/>) for [`struct file`](<https://elixir.bootlin.com/linux/v6.13.7/source/include/linux/fs.h#L1035>); the patch set uses something similar to [rcuref](<https://docs.kernel.org/RCU/rcuref.html>) with dead zones so that an unconditional increment can be done when taking a reference to an entry in the files table via the file descriptor. It provides a 3-5% performance increase when there is a lot of contention, ""which is great obviously"", but he thinks the scalability problem has just been pushed out further. 

So there is a need to explore other options. There was [a patch set implementing hazard pointers](<https://lwn.net/ml/all/20240917143402.930114-1-boqun.feng@gmail.com/>) for the kernel that he has been experimenting with, but that implementation is not suitable for `struct file`. It does scanning in the background and memory allocation. If hazard pointers were to be used, though, the file-reference path may need to allocate memory, which would add another possible error path. 

He would like to explore the idea further, but it is ""a very vague idea"" at this point; it might lead to regressions in the single-threaded case, though, which would not be desirable. Amir Goldstein asked where the scalability problem lies; Brauner said that it comes from contention with socket file descriptors in highly threaded workloads. ""It's not fantasized, it's actually an issue"", but is not likely to be one for highly threaded writes, because there are other synchronization activities present for those workloads. 

Wrapping up in his final 30 seconds, Brauner said that the Rust inode bindings should be discussed at some point. He plans to pick up those patches in the hopes of ""getting something like that merged"" within the next two years.
