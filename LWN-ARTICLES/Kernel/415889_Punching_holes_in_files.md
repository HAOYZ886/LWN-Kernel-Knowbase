---
title: Punching holes in files
url: https://lwn.net/Articles/415889/
date: "November 17, 2010"
category: fallocate
author: "By Jonathan Corbet November 17, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 17, 2010

The XFS and OCFS2 filesystems currently have the ability to "punch a hole" in a file - a portion of the file can be marked as unwanted and the associated storage released. Josef Bacik, noting that this capability may be added to other filesystems in the near future, came to the conclusion that the kernel should offer a standard interface for hole punching. The result is [an extension to the `fallocate()` system call](<https://lwn.net/Articles/415494/>) adding that ability. 

In particular, this patch adds a new flag (`FALLOC_FL_PUNCH_HOLE`) which is recognized by the system call. If the underlying filesystem is able to perform the operation, the indicated range of data will be removed from the file; otherwise `ENOTSUPP` will be returned. The current implementation will not change the size of the file; if the final blocks of the file are "punched" out, the file will retain the same length. There has been some discussion of whether changing the size of the file should be supported, but [the consensus seems to be](<https://lwn.net/Articles/415891/>) that, for now, changing the file size would create more problems than it would solve.
