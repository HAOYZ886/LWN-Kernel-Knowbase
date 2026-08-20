---
title: "LSFMM: User space NFS and CIFS servers"
url: https://lwn.net/Articles/548938/
date: "May 1, 2013"
category: CIFS
author: "By Jake Edge May 1, 2013 LSFMM Summit 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 1, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Boaz Harrosh and Jim Lieb led a discussion of kernel features needed by user-space filesystem servers ([NFS-GANESHA](<http://sourceforge.net/apps/trac/nfs-ganesha/>) and [Samba](<http://www.samba.org/>) in particular) at the 2013 LSFMM Summit. That was followed by an update on Samba from Steve French and Jeremy Allison. Harrosh started things off by saying there were eight or nine things that were needed from the kernel, but that no patches had yet been submitted for them. What he was looking for was agreement that the features would be accepted if reasonable code appeared. The discussion bogged down long before he reached the end of his list, however. 

Harrosh's first request was for an enhanced version of `readdir()`, though he thinks David Howells's [`xstat()`](<https://lwn.net/Articles/493400/>) system call is "too restrictive" and the model is not what he is looking for. He wants something that is more like the NFSv3 [READDIRPLUS](<http://tools.ietf.org/html/rfc1813#section-3.3.17>) operation, which returns multiple files with much more information than in a standard `readdir()`. He suggested having some kind of bitmap that described the kind of information the caller wanted returned (similar to `xstat()`), including things like extended attributes, access control lists, security context information, and so on. 

Dave Chinner was concerned about doing all that processing in the kernel. Mark Fasheh agreed, noting that user space already knows how to get all that kind of information. There is no reason to push the complexity into the kernel, Chinner said, but Harrosh and Lieb thought there was value in getting the information all at once. To Ted Ts'o that sounded like an assertion that a while loop in the kernel is somehow faster than one in user space, which he would need to see numbers on. 

Sage Weil suggested that `xstat()` be added first, then that machinery could be used to implement the extended `readdir()` along the lines of what Harrosh was looking for. Ts'o said that if a generic VFS fallback for filesystems that don't implement it was part of that work, there might be little complaint. 

Harrosh also mentioned some "very grave problems" with file locks because any close on the file will drop all of its locks. Ts'o noted that was a longstanding problem for which someone in POSIX needs to be punished. Harrosh suggested that there needed to be a way to lock on the file pointer to avoid this problem. 

Switching gears, Lieb said that the existing system calls used to impersonate client users are insufficient. When a user-space server is accessing a file on behalf of a user, it needs to change its UID, GID, alternate groups, LSM context(s), and so forth. Right now there are multiple system calls that must be made (`setfsuid()`, `setfsgid()`, `setgroups()`, ...), any of which can fail. He would like one call that either sets all of the user context pieces that are needed or fails and sets none of them. Unfortunately, the session ran out of time at that point, without any real conclusions or plans. 

#### Samba update

The kernel's CIFS client has seen 380 changes over the last year, French said. That is mostly work on SMB2, but also some work on the beginnings of SMB3. There have been a lot of Red Hat stability fixes as well. The big news in the Samba world is that Samba 4 was released after seven years. 

Allison said that there are a bunch of new features in the server, but the biggest missing piece is [rich access control lists](<http://en.wikipedia.org/wiki/Richacls>) (ACLs). The project has a module that stores Windows ACLs as filesystem extended attributes, which can be mapped to POSIX ACLs, he said. Microsoft has added the new [ReFS](<http://msdn.microsoft.com/en-us/library/windows/desktop/hh848060%28v=vs.85%29.aspx>) filesystem that doesn't support "named streams", Allison said, so Samba can eventually get rid of them. That's good because "virus writers were the only ones who used named streams", he said. Samba has yet to implement advanced leasing, and he doesn't yet know what Windows does there. For the future, the plans are to fully comply with SMB3, but to do so by not implementing any of the optional features. Almost all of the 3.0 features are optional, Allison said, except encryption, which is already supported by Samba.
