---
title: XFS parent pointers
url: https://lwn.net/Articles/753480/
date: "May 7, 2018"
category: "Filesystems-XFS"
author: "By Jake Edge May 7, 2018 LSFMM"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 7, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

At the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Allison Henderson led a session to discuss an XFS feature she has been working on: [parent pointers](<https://lwn.net/Articles/752308/>). These would be pointers stored in extended attributes (xattrs) that would allow various tools to reconstruct the path for a file from its inode. In XFS repair scenarios, that path will help with reconstruction as well as provide users with better information about where the problems lie. 

[ ![\[Allison Henderson\]](https://static.lwn.net/images/2018/lsf-henderson-sm.jpg) ](<https://lwn.net/Articles/753478/>)

The patch set has had a "bumpy history", she said. Lots of issues were identified with earlier versions of the patch set, which have now been addressed. Historically there were problems with locking order, but now the goal is to not have to lock the parent inode when creating the parent pointer. The xattr name will be the parent inode number and generation, along with the directory offset of the file. The xattr value will be the file name. 

Jeff Layton said he sees how it would be useful to be able to walk the tree back to the root to recreate the path, but wondered about hard links. Dave Chinner said that each link would create its own parent pointer attribute. Al Viro asked about rename operations during the tree walk, but Chinner said there is no real problem there. The walk is done in user space (using `ioctl()` calls); the idea is that if there is problem in inode X, sector Y, a reverse lookup can be done to provide the user with the path. If the path changes during the walk, the user-space program should redo it. 

Henderson said that one use case is for online scrub and repair. It will allow inodes that have been orphaned to be reconnected correctly. The error reporting will also be better because there will be a path associated with the inode where problems were found. She is trying to gather information on other use cases so that she can ensure that the feature supports them. Chinner said that filesystem repair is an important use; simply dumping a million files into the `lost+found` directory is useless. 

Ted Ts'o asked about the performance of the feature. Chinner said it simply added an xattr operation to each file create, rename, link, and unlink operation. That should be fine if the xattr fits in the inode, Ts'o said, but Chinner noted that xattrs are being used everywhere these days, so xattr operations are generally expected.
