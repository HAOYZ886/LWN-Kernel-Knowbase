---
title: The return of write barriers
url: https://lwn.net/Articles/77074/
date: "March 24, 2004"
category: Block layer; Write barriers
author: 
---

One of the tasks on the 2.5 "to do" list was the implementation of proper write barriers in the block I/O subsystem. Any code which attempts to implement true transactional behavior on disk-based files needs this capability. Without it, systems like journaling filesystems and database managers lack the control they need over the order in which data is written to disk. Mis-ordered writes can lead to data corruption and other unfortunate things. 

The 2.6 block I/O subsystem was designed with barrier support as a core feature. But, at this point, most low-level block drivers do not actually implement barriers, and the filesystems do not use them. Patches to fill in some of the gaps have been around for a while (LWN looked at barriers [last October](<http://lwn.net/Articles/54070/>)), but have not yet been merged. 

There has been a new surge of interest in proper barrier support, perhaps as a result of applications vendors starting to take a hard look at the 2.6 kernel. Now Jens Axboe and Chris Mason have put together [a new barrier support patch](<http://lwn.net/Articles/76540/>) which gets Linux closer to being able to provide real disk I/O guarantees. With this patch, write barriers work, but only on IDE drives (not SCSI or serial ATA), and only with the reiserfs and ext3 filesystems. Even then, things are qualified: ""ext3 works but only if things don't go wrong."" 

In other words, barrier support will be staying on the "to do" list for a little while longer yet. But the work is being done, and 2.6 should be able to implement real barriers before it is all over.
