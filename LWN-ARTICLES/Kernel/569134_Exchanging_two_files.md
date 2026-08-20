---
title: Exchanging two files
url: https://lwn.net/Articles/569134/
date: "October 2, 2013"
category: renameat2
author: "By Jonathan Corbet October 2, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 2, 2013

The [`renameat()` system call](<http://man7.org/linux/man-pages/man2/renameat.2.html>) changes the name of the file given as an argument, possibly replacing an existing file in the process. This operation is atomic; the view of the filesystem from user space will reflect the situation before or after the `renameat()` call, but it will never expose an intermediate state. Things work well when one file is involved, but what happens when multiple rename operations need to be run as a single atomic operation? That is a big problem, but, thanks to a patch from Miklos Szeredi, we might have a solution to a smaller subset. 

The problem Miklos is trying to solve is the problem of exchanging two files — both files continue to exist, but their names have been swapped. To achieve this, he has posted [a patch set](<https://lwn.net/Articles/569028/>) adding a new `renameat2()` system call: 

```
int renameat2(int olddir, const char *oldname, 
    		  int newdir, const char *newname, unsigned int flags);
```

This system call differs from `renameat()` in that it has the new `flags` argument; if `flags` is zero, `renameat2()` behaves exactly as `renameat()`. If, instead, `flags` contains `RENAME_EXCHANGE`, an existing file at `newname` will not be deleted; instead, it will be renamed to `oldname`. Thus, with this flag, `renameat2()` can be used to atomically exchange two files. The main use case for `renameat2()` is to support union filesystems, where it is often desirable to atomically replace files or directories with "whiteouts" indicating that they have been deleted. One could imagine other possibilities as well; Miklos suggests atomically replacing a directory with a symbolic link as one of them. 

No review comments have been posted as of this writing.
