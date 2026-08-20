---
title: Trimming down sysfs
url: https://lwn.net/Articles/69523/
date: "February 4, 2004"
category: Sysfs
author: 
---

The sysfs virtual filesystem is one of the many additions to the 2.6 kernel. sysfs is the user-space presentation of the kernel's device model; it is used by the `udev` utility to create device nodes for hardware and, eventually, numerous other purposes. There is a lot of information about the system available under sysfs; it may, eventually, replace many of the files currently found under `/proc`. 

There is one little problem with sysfs, however. It is built as a simple kernel filesystem using the VFS cache as its backing store. This is an easy way to build a kernel filesystem, since the generic VFS code does most of the hard work for you. It does, however, require the kernel to maintain a directory entry ("dentry") cache entry and an inode in memory for every file and directory in the filesystem. As sysfs has grown, the amount of memory it dedicates to dentries and inodes has grown as well. Even a small system can have several hundred files in `/sys`; that number can grow impressively for larger systems. The memory that all those sysfs nodes occupy can be painful for very small systems (which do not have much memory to spare) and for very large systems (because sysfs lives in low memory, which is at a premium). 

In order to deal with this problem, Maneesh Soni has been working on a set of patches which provides a true backing store for sysfs. These patches (the full set can be found in the "patches and updates" section, below) retain the current VFS-level cache for directories; doing otherwise turns out to open a fairly large can of worms in how the device model and the VFS interact. All of the attribute files (which make up 70% or so of sysfs entries), however, can be more compactly represented by the sysfs code itself. All that is really needed for an attribute, after all, is its name and pointers to the "show" and "store" functions. 

To this end, the patches create a new `sysfs_dirent` structure which describes a node in the sysfs hierarchy. These structures implement an in-core representation of the sysfs tree that takes up far less space than the full VFS-cached version. When user space accesses a specific attribute node, it is a fairly straightforward matter to create the inode and dentry structures on the spot. Neither structure need be pinned into memory, so they can be aged out with the rest of the VFS cache. 

The result of all this work, Maneesh [claims](<https://lwn.net/Articles/69612/>), is a savings of 145MB of low memory on his (massive) test system. The number of active dentries in this system drops from over 60,000 to under 9,000. Unlike early versions of this patch, the current effort also avoids making changes to the `kobject` structure, so no penalty is paid for structures using kobjects which do not appear in sysfs. As the patch has evolved, the number of criticisms has gone down; sysfs backing store appears to be getting closer to ready for inclusion.
