---
title: A system call for unsharing
url: https://lwn.net/Articles/135321/
date: "May 10, 2005"
category: Namespaces; unshare
author: 
---

When a new process is created with the `clone()` system call, a set of flags is provided which tells the kernel which resources, if any, should be shared between that process and its parent. Potentially shareable resources include virtual memory, open files, signal handlers, and more. New processes also share, by default, the filesystem namespace seen by their parent (and, usually, by the system as a whole). 

In the current Linux kernel, the sharing decisions made at `clone()` time last for the lifetime of the processes involved. There is not usually a reason to change resource sharing, but recent discussions on supporting private mounts (with the filesystems in user space patch, or otherwise) have suggested that it would actually be useful for a process to be able to "unshare" resources after its creation. In particular, if a process could detach itself from the global filesystem namespace and create its own, it would be possible to set up that new namespace with whatever private mounts that process needs. If this functionality were used within a PAM module, it would be relatively easy for administrators to set up per-user views of the filesystem, complete with private mounts. 

To that end, Jenak Desai has posted [a patch](<https://lwn.net/Articles/135266/>) adding a new `unshare()` system call. The interface is simple enough: 

```
long unshare(unsigned long flags);
```

The `flags` argument can be `CLONE_NEWNS` (to create a new filesystem namespace), `CLONE_VM` (to establish a private virtual address space) or `CLONE_SIGHAND` (to unshare signal handlers). If all goes well, when the call returns, the designated resource(s) will now be private to the calling process; otherwise the situation is unchanged. 

This patch has not yet made it to the linux-kernel mailing list, and may see some changes before it is considered for inclusion.
