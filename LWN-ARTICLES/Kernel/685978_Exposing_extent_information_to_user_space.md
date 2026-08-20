---
title: Exposing extent information to user space
url: https://lwn.net/Articles/685978/
date: "May 4, 2016"
category: Filesystems
author: "By Jake Edge May 4, 2016 LSFMM 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
May 4, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

In a short, filesystem-only session at the 2016 Linux Storage, Filesystem, and Memory-Management Summit, Josef Bacik led a discussion on exposing information on extents, which are contiguous ranges of blocks allocated for a file (or files) by the filesystem, to user space. That could be done either by extending the [`FIEMAP` `ioctl()` command](<https://lwn.net/Articles/260795/>) or by coming up with a new interface. Bacik said that he was standing in for Mark Fasheh, who was unable to attend the session. 

[ ![\[Josef Bacik\]](https://static.lwn.net/images/2016/lsf-bacik-sm.jpg) ](<https://lwn.net/Articles/686115/>)

`FIEMAP` just reports whether an extent is shared or not, but there are some applications that want to know which inodes are sharing the extents. There are reserved 64-bit fields in `struct fiemap_extent` that could be used to report the inode numbers, Bacik said. He asked if that seemed like a reasonable approach. 

Ric Wheeler wondered if there was really a need for applications to unwind all of this information. He asked: "Is there a backup application that will use this?" Jeff Mahoney responded that there is someone requesting the functionality. 

Darrick Wong said that as part of his [reverse mapping and `reflink()` work for XFS](<https://lwn.net/Articles/684826/>) he has an interface that will allow applications to retrieve that kind of information. You can pass a range of physical block numbers to the reverse-map `ioctl()` and get back a list of objects (e.g. inodes) that think they own those blocks, he said. 

Bacik said that sounded like the right interface: "Let's use that." Wong said that he would post some patches once he returned home from the summit.
