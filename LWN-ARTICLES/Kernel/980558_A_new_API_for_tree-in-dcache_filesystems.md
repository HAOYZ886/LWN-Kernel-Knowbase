---
title: "A new API for tree-in-dcache filesystems"
url: https://lwn.net/Articles/980558/
date: "July 9, 2024"
category: Filesystems
author: "By Jake Edge July 9, 2024 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
July 9, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

There are a number of kernel filesystems that store their directory entries directly in the directory-entry cache (dcache) without having any permanent storage for those objects. It started out as a "neat hack" for [ramfs](<https://docs.kernel.org/filesystems/ramfs-rootfs-initramfs.html>), Al Viro said, at the start of his filesystem-track session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). Unfortunately, as the use of this technique has grown into other filesystems, there has been a lot of scope creep that has gotten out of control. He wanted to discuss some new infrastructure that he is working on to try to clean some of that up. 

[ ![\[Al Viro\]](https://static.lwn.net/images/2024/lsfmb-viro-sm.png) ](<https://lwn.net/Articles/981165/>)

Viro displayed some [notes](<https://ftp.linux.org.uk/pub/people/viro/notes>) on his thoughts to accompany his talk; some of this article derives from those notes. He has a patch set to implement those ideas (contained in his ["untested.persistency" branch](<https://git.kernel.org/pub/scm/linux/kernel/git/viro/vfs.git/log/?h=untested.persistency>)) that is "very much a work in progress", which is untested and may not compile on anything other than x86, he said. He wanted to describe the problem it is meant to solve and how it does so. 

It all started with a demonstration by Linus Torvalds of how to create a filesystem without a backing store, he said. The technique kept all of the files and directories in the cache and was the basis for ramfs. A "controlled dentry [directory entry] leak" was used; reference counts are artificially increased to ensure that the directory entries do not get evicted. When an unmount is done, they are all cleaned up. 

The technique was adopted by tmpfs, hugetlbfs, and in other places, because it is simpler than what procfs uses. There are problems that arise in some of the other users, however, that do not exist in the original. The original intent was only for filesystems that were being populated from user space, but eventually it was used for filesystems that are populated by the kernel, or, perhaps worse, both the kernel and user space. 

For example, [`rmdir()`](<https://man7.org/linux/man-pages/man2/rmdir.2.html>) only removes directories that are empty, but the configfs user-space tools expect the system call to be able to remove a populated subtree if all of its entries were created by the kernel. If there are user-created subdirectories, the tools expect the `rmdir()` to fail. Christian Brauner pointed out that the control-group filesystem (cgroupfs) also has this behavior. Viro said that filesystems of this sort have to implement their own `rmdir()` because it is so specialized. For configfs, it needs to check if there have been any directories created by the user inside the target—or any that are in progress. The code "is horrible", he said. 

There is a real need for some infrastructure to help these filesystems, Viro said. There are around a dozen different implementations of the subdirectory-removal handling, none of which have been done correctly. His idea is to introduce a flag, `DCACHE_PERSISTENT` that will be used to mark the dentries that are being "controllably leaked" so that they can be properly handled. Then the kernel-initiated operations and those from user space can set the flag, so that they are handled in the same way, which is not the case right now. 

There would be two new functions that would be the counterparts to [`dget()`](<https://elixir.bootlin.com/linux/v6.9.8/source/include/linux/dcache.h#L312>) and [`dput()`](<https://elixir.bootlin.com/linux/v6.9.8/source/fs/dcache.c#L824>) (which obtain and release references to directory entries); `d_make_persistent()` would do the equivalent of `dget()` and set the flag, while `d_make_discardable()` will do a `dput()` and clear it. There are new helper functions to handle both the simple filesystems like ramfs and the more complicated varieties, including handling some of the variations of open-coded directory-removal code. There are more details in the notes file. 

There are still four filesystems that remain to be converted, Viro said. They all have "interesting problems" that need to be resolved; two of them are for [USB gadgets](<https://www.kernel.org/doc/html/v4.17/driver-api/usb/gadget.html>), one is configfs, and the other is apparmorfs. The diffstat of his patch set shows that the changes would actually result in a net removal of around 500 lines of code from the tree. 

He had hoped to discuss configfs with Christoph Hellwig, who was not present, though he did arrive later in the day. He plans to talk to Greg Kroah-Hartman about the USB-gadget filesystems, but is not sure who to talk to about apparmorfs. There is some strange locking being done in apparmorfs, which he mentioned to the AppArmor developers, but got nowhere, he said. There was some further discussion on this and related work as time ran out on the session.
