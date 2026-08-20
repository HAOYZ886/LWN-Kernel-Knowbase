---
title: Time for ext4
url: https://lwn.net/Articles/190169/
date: "July 5, 2006"
category: "Filesystems-ext4"
author: 
---

A few weeks ago, this page [looked at possible additions to the ext3 filesystem](<http://lwn.net/Articles/187336/>) and the question of whether the time had come to freeze ext3 and put new features into a new ext4 filesystem again. The ext2/3 filesystem developers have now [responded to that discussion](<http://lwn.net/Articles/189950/>) with a clear answer: they will be moving on to ext4. 

More specifically, a new filesystem will be created under `fs/ext4` in the kernel source. Said filesystem will register itself as "`ext3dev`," in an attempt to make it crystal clear that it is a development filesystem, not suitable for the storage of data which one actually wishes to keep. New feature work - especially changes which change on-disk formats and prevent interoperation with current ext3 implementations \- will go into this new filesystem, while ext3 will continue to receive bug fixes and some safe improvements. Throughout this process, the new filesystem will retain its ability to work with the current ext3 format. 

Sometime in the future, ext3dev will be declared stable and renamed "ext4." Once the last bugs have been shaken out, this filesystem will lose its "experimental" designation and users will be encouraged to upgrade. Since support for ext3 formats will be there, this upgrade should be an easy process, with no backup-and-restore step or downtime required. Further in the future, the ext3 code may be removed and ext4 would transparently handle ext3 filesystems as well. 

There seems to be little opposition to this approach, so it would appear that things will happen this way. Since the addition of a new, experimental filesystem carries little regression risk, the creation of ext4 and the addition of some new features (extents, for example) could yet happen for 2.6.18.
