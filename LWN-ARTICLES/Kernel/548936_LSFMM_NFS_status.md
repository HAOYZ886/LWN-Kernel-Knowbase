---
title: "LSFMM: NFS status"
url: https://lwn.net/Articles/548936/
date: "May 1, 2013"
category: "Filesystems-NFS; Network filesystems"
author: "By Jake Edge May 1, 2013 LSFMM Summit 2013"
---

By **Jake Edge**  
May 1, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Trond Myklebust and J. Bruce Fields opened the 2013 LSFMM Summit's Filesystem-only track with an update on the status of NFS. On the client side, NFS 4.1 is a "done deal at this point", Myklebust said, and he has spent the last few months finding the "last few corner-case deadlocks". There are a few patches for the 3.9 kernel and a few more queued for 3.10, but it is basically done. 

Development on the NFS client has moved over to NFS 4.2, where one of the biggest new features is [copy offload](<https://lwn.net/Articles/548347/>). The main thing holding that back is the lack of a new system call. Jeremy Allison noted that Samba is already doing copy offload using a Btrfs `ioctl()`. If all goes well, copy offload will be merged in the next six months or so, Myklebust said. 

The other big 4.2 feature is [labeled NFS](<http://selinuxproject.org/page/Labeled_NFS>), which adds SELinux mandatory access control (MAC) to NFS. The main application is for virtualization, and the KVM users want to use labeled NFS to improve security on systems shared by multiple virtual machines. Myklebust said he had hoped to have it ready for the 3.10 merge window, but there were still "some niggles". It should be in 3.11, he said, "unless we are doing something really wrong". 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Myklebust also gave a plug for the [NFSometer](<http://fedoraproject.org/wiki/Features/NFSometer>) tool, which takes statistics from `/proc` and "makes fancy graphs". It is useful to visualize the NFS performance of systems. It can also compare multiple sets of stats in bar charts to help users compare performance between various NFS versions (e.g. v3 vs. v4) as well as between Linux kernel versions. 

On the server side, Fields said that 4.0 is "ancient history" at this point, while the 4.1 RFC has been out for four or five years. 4.1 has parallel NFS (pNFS) and a bunch of mandatory features that are mostly invisible to users. 4.2 is what the working group is polishing off right now. It contains copy offload, labeled NFS, hole detection and punching, as well as a "non-`fallocate()`-compatible space reservation". All of the substantive features are optional in 4.2, Fields said. 

For Linux, the NFS server code has had 4.1 support for some time; it became the default in the 3.9 kernel. No optional features, including pNFS, are available, though. Fields would like to start working on 4.2 features like labeled NFS soon. 

Fields said that he is trying to get VFS support for [delegations](<http://docs.oracle.com/cd/E19253-01/816-4555/rfsrefer-140/index.html>) (which are somewhat similar to leases) merged. The difficulty is in getting cycles from Al Viro, Linus Torvalds, and "whoever else" to review the code. A delegation implementation was tried for ext4, but there were serious performance implications. 

The [problem](<https://lwn.net/Articles/544298/>) with ext4 `readdir()` cookies and GlusterFS was also mentioned by Fields. A fix has been put in for GlusterFS on ext4 that is rather fragile, Ted Ts'o said. Zach Brown called it a "just for now fix". The basic problem is GlusterFS's need for some extra bits in the directory offset to store the ID of the "brick" where the file resides, Fields said. The infinite lifetime of `readdir()` cookies is also part of the problem, Ts'o said. Allison pointed out that Samba had the same problem for DOS, but the solution there was to require an update to Windows 95\. 

The last topic was [rich access control lists](<http://en.wikipedia.org/wiki/Richacls>) (ACLs). Fields said that the developers know how to add rich ACLs, but a volunteer is needed to look at the code and review comments from Christoph Hellwig and others. It is something that is "totally doable". Allison cautioned that Windows ACLs are different from NFSv4 ACLs, even though the NFS version was based on the specification documented by Microsoft. Unfortunately, the specification is not what Microsoft actually implemented, so a mapping between them will be required.
