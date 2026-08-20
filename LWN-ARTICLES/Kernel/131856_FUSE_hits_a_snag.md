---
title: FUSE hits a snag
url: https://lwn.net/Articles/131856/
date: "April 12, 2005"
category: "Filesystems-In user space"
author: 
---

The filesystems in user space (FUSE - covered here [in January, 2004](<https://lwn.net/Articles/68104/>)) provides a kernel interface and library which makes it easy to implement filesystems with a user-space process. With FUSE, a user can mount a filesystem contained with a tar archive, implemented via an FTP session, or "tunneled" from a remote system via ssh. It is a powerful tool with many users, and its authors have been pushing for inclusion into the mainline kernel for some time now. That merge has been delayed pending a review of the patch by a few interested developers. 

That review has happened, and it has turned up a problem; it seems that FUSE, in some situations, implements some rather strange filesystem semantics. 

Consider the case of a filesystem hosted in a tar archive. FUSE will present files within the archive with the owners and permission modes specified inside that archive. The owner and permissions of the files, in other words, do not necessarily have anything to do with the owner of the archive or the user who mounted it as a filesystem. To allow that user to actually work with files in the archive, the "tarfs" FUSE module disables ordinary permissions checking. A file may, according to a tool like `ls`, be owned by another user and inaccessible, but the user who mounted the filesystem has full access anyway. FUSE also ensures that no other user has _any_ access to the mounted filesystem - not even root. 

This twisting of filesystem semantics does not sit well with some kernel developers, who tend to think that Linux systems should behave like Linux. The FUSE semantics have the potential to confuse programs which think that the advertised file permissions actually mean something (though, evidently, that tends not to be a problem in real use) and it makes it impossible to mount a filesystem for use by more than one user. So these developers have asked that the FUSE semantics be removed, and that a FUSE filesystem behave more like the VFAT-style systems; the user mounting the filesystem should own the files, and reasonable permissions should be applied. 

In fact, FUSE does provide an option ("`allow_others`") which causes it to behave in this way. But that approach goes against what FUSE is trying to provide, and raises some security issues of its own. FUSE hacker Miklos Szeredi [sees the issue this way](<https://lwn.net/Articles/131858/>): 

I want the tar filesystem to be analogous to running tar. When I run tar, other users are not notified of the output, it's only for me. If they want to run tar, they can too. The same can be true for tarfs. I mount it for my purpose, others can mount it for theirs. Since the daemon providing the filesystem always runs with the same capabilities as the user who did the mount, I and others will always get the permissions that we have on the actual tar file. 

In this view, a FUSE filesystem is very much a single-user thing. In some cases, it really should be that way; consider a remote filesystem implemented via an `ssh` connection. The user mounting the filesystem presumably has the right to access the remote system, on the remote system's terms. The local FUSE filesystem should not be trying to figure out what the permissions on remote files should be. Other users on the local system - even the root user - may have no right to access the remote system, and should not be able to use the FUSE filesystem to do so. 

It's not clear where this discussion will go. There are some clear reasons behind the behavior implemented by FUSE, and it may remain available, though, perhaps, not as a default, and possibly implemented in a different way. The little-used Linux namespace capability has been mentioned as a way of hiding single-user FUSE filesystems, though there may be some practical difficulties in making namespaces actually work with FUSE. Until the core filesystem hackers are happy, however, FUSE is likely to have a rough path into the mainline.
