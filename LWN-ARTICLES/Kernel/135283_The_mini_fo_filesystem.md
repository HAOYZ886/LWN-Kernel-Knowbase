---
title: The mini_fo filesystem
url: https://lwn.net/Articles/135283/
date: "May 10, 2005"
category: Filesystems
author: 
---

Markus Klotzbuecher recently [announced](<https://lwn.net/Articles/135158/>) the release of mini_fo 0.6.0. Mini_fo provides (what has been called in other systems) a "translucent" or "copy on write" filesystem. A read-only, base filesystem (possibly from a remote system or CDROM) can be made to appear, via mini_fo, as a local, writable filesystem. This functionality is useful for sharing filesystems with local overrides, live CD systems, sandboxing applications, and more. 

At its core, mini_fo performs a simple fan-out operation. Each inode, dentry, and file structure associated with a mini_fo filesystem contains (via its private data) pointers to two other structures of the same type. One of them refers to the file or directory on the base filesystem; the other, instead, is for a local version of the file or directory on a local "storage filesystem." Both are hidden from user space, which thinks it is dealing directly with a file stored in the mini_fo filesystem. 

When a mini_fo filesystem is first created, it appears as an exact copy of the underlying base filesystem. Any operation which reads files or directories is simply passed through to the base filesystem, with almost no additional overhead. In this mode, mini_fo functions as a sort of loopback filesystem. 

Things change, however, when a file is opened for writing. In this case, mini_fo will create a copy of the file on the storage filesystem, with all of the data moved over. Any subsequent operations on that file will used the locally-stored version rather than the base version. So any changes made will appear locally, but they will not be propagated back to the base. Changes will be persistent across mounts as long as the storage directory used by mini_fo is not modified by anything except mini_fo. 

Modified files are not the full story, of course; mini_fo must also cope with operations like deletes and renames. To that end, it maintains a set of lists of files which it knows about locally; there is one list for modified files, one for deleted files, one for files created locally, etc. These lists are stored in-kernel as standard linked lists. They are also written to the storage filesystem in a magic file (named `META_dAfFgHE39ktF3HD2sr`, for what it's worth) and reloaded from that file when the filesystem is mounted. 

This release of mini_fo works with both the 2.4 and 2.6 kernels. Its author claims that it is intended for use with embedded systems, and thus has a small memory footprint. See [the mini_fo web page](<http://www.denx.de/twiki/bin/view/Know/MiniFOHome>) for more information.
