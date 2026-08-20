---
title: In Brief
url: https://lwn.net/Articles/344038/
date: "July 29, 2009"
category: fanotify
author: "By Jonathan Corbet July 29, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 29, 2009

**FAT timestamps**. The FAT filesystem has a number of deficiencies. The fact that it cannot record time stamps for the root directory of a filesystem is probably not at the top of most peoples' lists, but Jorg Schummer has put together [a patch](<http://lwn.net/Articles/343565/>) to provide those time stamps anyway. The patch is a hack which stores the time stamp information in the FAT volume label, essentially hiding it from any system which doesn't know to look for it. This is not a new scheme; Mac OS X does the same thing. There does not seem to be a great clamor for this feature, but it is optional, the implementation is straightforward, and it's off by default. So there is little reason to leave it out either. 

**Remapping ext2/3 UIDs**. Another failing of FAT is its inability to associate user or group ownership information with files. One would not normally want to port this "feature" to more complete filesystems, but Ludwig Nussel has [noted a problem](<http://lwn.net/Articles/343024/>): a user moving an ext3 filesystem from one system to another will have problems accessing the files if said user's accounts have different user IDs on the two boxes. The solution is to add a `uid=` mount option to ext2 and ext3; the filesystem will then map between the given user ID (on the running system) and zero (on the filesystem). 

There doesn't seem to be a great clamor for this feature either; the use of ext3 on filesystems moved between machines is probably relatively rare. Still, Andreas Dilger [indicated](<https://lwn.net/Articles/344054/>) that the feature might have its uses, but that some changes would be welcome. The ability to create root-owned setuid files needed to go away, and it would be nice to have a more general "remap UID1 to UID2" capability instead of just mapping to and from the root UID. Andreas also requested an ext4 version of the patch. 

**Fanotify**. Eric Paris has posted [a description of the new fanotify API](<http://lwn.net/Articles/343346/>) for comments, noting that real patches will follow soon. That API has changed considerably since it was [covered here](<http://lwn.net/Articles/339399/>) at the beginning of July; the strange use of `getsockopt()` to get notifications is no more. Instead, a relatively normal socket is created, with `read()` being used to read notification events. There were a number of comments and suggestions, but the consensus seems to be that things are headed in the right direction. 

**ABUSE**. We have FUSE, which allows the implementation of filesystems in user space, and CUSE, which does the same for char devices. So why not do the same thing for block devices? With Zachary Amsden's [ABUSE patch](<http://lwn.net/Articles/343514/>), that now becomes possible. Zachary says: ""This device is not about performance, is it about extending the boundaries of the kernel to the almost improbable."" The code commentary notes that the feature can be "incredibly useful," but it's not clear what use case is being targeted at the moment. 

ABUSE is highly unlikely to be merged, for the simple reason that much of what it does is already doable with the network block device (NBD) driver. Zachary [plans to move to NBD](<https://lwn.net/Articles/344067/>) for whatever purpose he has in mind. That purpose, apparently, makes it necessary to have access to partitions, which is why FUSE cannot be used. 

The partitions topic led to a small side discussion, where Alan Cox [suggested](<https://lwn.net/Articles/344068/>) that partition support should be removed from the kernel altogether. Instead, the device mapper should be used to implement partitions. There are a lot of advantages - mostly administrative flexibility - which come from the use of the device mapper, but there are users, [Linus included](<https://lwn.net/Articles/344069/>), who are not interested in requiring its use. So the kernel's partition code will not be going anywhere anytime soon. 

**A new book on the way**. Man pages maintainer Michael Kerrisk, while writing about a recent release, noted that he is well along in the writing of a new book which extensively documents the Linux kernel's user-space API. It will not be light reading; it looks to end up at about 1500 pages. For the curious, Michael has posted [a general description of the book](<http://blog.man7.org/2009/07/whats-book-about.html>) and [the table of contents](<http://blog.man7.org/2009/07/64-chapters.html>). Publication is expected sometime in the first half of 2010.
