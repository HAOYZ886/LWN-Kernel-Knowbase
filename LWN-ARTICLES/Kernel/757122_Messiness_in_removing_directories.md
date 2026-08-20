---
title: Messiness in removing directories
url: https://lwn.net/Articles/757122/
date: "June 13, 2018"
category: "Filesystems-Virtual filesystem layer"
author: "By Jake Edge June 13, 2018 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
June 13, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

In the filesystem track at the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Al Viro discussed some problems he has recently spotted in the implementation of [`rmdir()`](<http://man7.org/linux/man-pages/man2/rmdir.2.html>). He covered some of the history of that implementation and how things got to where they are now. He also described areas that needed to be checked because the problem may be present in different places in multiple filesystems. 

The fundamental problem is a race condition where operations can end up being performed on directories that have already been removed, which can lead to some rather "unpleasant" outcomes, Viro said. One warning, however: it was a difficult session to follow, with lots of gory details from deep inside the VFS, so it is quite possible that I have some (many?) of the details wrong here. Since LSFMM there has been no real discussion of the problem and its solution on the mailing lists that I have found. 

Viro said that some reports from the syzkaller fuzzer bot (syzbot) just prior to the summit had started him looking at `rmdir()`. The easiest way to trigger the problem syzbot found is to remove a directory with an enormous directory entry (dentry) tree in the cache. The call will fail because the directory is not empty but in the process it will call [`shrink_dcache_parent()`](<https://elixir.bootlin.com/linux/v4.17/source/fs/dcache.c#L1477>) for historical reasons. The code previously checked that the directory inode reference count was one and return `EBUSY` if it was not. It was an easy check that would prevent anyone from creating an entry in the directory after it was deleted, which could lead to filesystem corruption. 

But then the dentry cache (dcache) was added; there was no longer a reference to the inode for a cached reference to the directory dentry. The test could change to check the dentry instead of the inode, but negative dentries would have references to the directory dentry, which would make the test fail. The solution to that was to try to evict child dentries from the cache before doing the check. It was done after the check to ensure the directory is empty, but there is still a race. 

The ext2 filesystem added a step where it set the victim's `i_size` to zero, which would allow removing the directory even when it was busy. Around the beginning of the 2.4 era, Viro got "sufficiently annoyed" by races around directory removal that he lifted the ext2 solution into the VFS layer. Instead of change `i_size`, though, his code would just mark the victim while it was locked. All of the filesystem primitives would then check that the directory was not marked dead before operating on it. 

Around 2011, it was noticed that the dcache could still have negative dentries for children and a positive dentry for the directory itself after it had been removed. The obvious solution was to use `shrink_dcache_parent()` and to remove the directory dentry after an `rmdir()`. It turned out that [`rename()`](<http://man7.org/linux/man-pages/man2/rename.2.html>) had a similar case with the exact same problems, he said. 

The "real mess" that he has spotted recently has to do with removing a directory on a special filesystem (e.g. configfs, debugfs) if something is mounted on it. It used to be that a directory with something mounted on it could not be removed, but the container folks complained about that. One container could block many others from cleaning up by making a directory in a shared filesystem and then mounting something on it. That was changed so that the directory can be deleted, but doing so leaks a `struct vfsmount` object. 

It is not just `rmdir()` that is affected or it could simply be fixed there. For example, `write()` has "no idea this kind of thing is possible". It affects other filesystems too, including sysfs, selinuxfs, and apparmorfs, but not procfs. 

`rmdir()` and `rename()` obviously need to be fixed, Viro said. He looked at NFS and thinks it does not suffer from this problem, but he is not sure about CIFS or AFS (and said he doesn't even want to think about ncpfs). The 4.18 merge window should clear up the ncpfs problem, since that filesystem was removed from the kernel as part of the staging tree pull. Viro hopes to get the cluster filesystem developers looking at those. He also asked that filesystem developers check that all of their filesystem's operations (`ioctl()`, `chmod()`, ...) will not operate on a directory that has been removed.
