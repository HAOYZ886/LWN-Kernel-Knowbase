---
title: Online resizing of ext3 filesystems
url: https://lwn.net/Articles/89560/
date: "June 15, 2004"
category: "Filesystems-ext3; Resizing filesystems"
author: 
---

One of the patches which slipped into 2.6.7-rc3-mm2 is one by Andreas Dilger and others which makes it possible to resize a running ext3 filesystem on the fly. This patch has been shipped with Fedora kernels for a little while, but has not seen a lot of wider use. That could change, of course, if the resize patch finds its way into the mainline. 

The resize patch is conceptually quite simple. It simply adds one or more block groups which make use of extra space which, one hopes, is sitting there idle at the end of the existing filesystem. Once the block groups are hooked into the filesystem data structures, a simple `ioctl()` call or remount will make the space available. Behind this apparent simplicity, of course, is a significant amount of code which makes the resize operation happen on a modern, complex filesystem in a robust manner. 

People wanting to try out resizing will need a few things: 

  * A kernel (such as 2.6.7-rc3-mm2) with the online resize patch included. 

  * A patch to e2fsprogs to make use of the resize capability; it is available [from the ext2resize SourceForge download area](<http://sourceforge.net/tracker/?atid=303834&group_id=3834&func=browse>). 

  * Free disk space into which the filesystem can expand. Usually this means that the filesystem should live in a device mapper partition which can be expanded as well. 

  * A very good backup of your filesystem. 

This patch and its associated documentation (or lack thereof) still require some work before being ready for widespread deployment. Once they get there, however, life should get easier for system administrators who, throughout history, have routinely found out that all that "extra space" they figured into their filesystems is never enough.
