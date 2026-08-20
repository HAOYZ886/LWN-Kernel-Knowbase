---
title: Fixing error reporting—again
url: https://lwn.net/Articles/752613/
date: "April 25, 2018"
category: "Block layer-Writeback"
author: "By Jake Edge April 25, 2018 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
April 25, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

After a [session](<https://lwn.net/Articles/718734/>) at last year's Linux Storage, Filesystem, and Memory Management Summit (LSFMM), Jeff Layton was able to [make some improvements](<https://lwn.net/Articles/724307/>) to block-layer error handling. Those changes, which [added](<https://lwn.net/Articles/724232/>) a new `errseq_t` type to hold an error number and sequence number, seemed to help and were well received—except [by the PostgreSQL developers](<https://lwn.net/Articles/752063/>). So Layton led a session at the 2018 LSFMM to discuss ways to improve things further; it would be followed later in the week with a session by one of the PostgreSQL developers to look at the specifics of the problem from their perspective. 

[ ![\[Jeff Layton\]](https://static.lwn.net/images/2018/lsf-layton-sm.jpg) ](<https://lwn.net/Articles/752726/>)

Layton started by noting that Matthew Wilcox had come up with a [patch](<https://lkml.org/lkml/2018/4/23/994>) to restore the behavior that PostgreSQL relies on, which is to receive I/O errors from periodic `fsync()` calls made by a checkpointing process. However, that change still could not guarantee that errors would always be reported; if the inode is evicted from memory, the error stored there will be lost. The lack of a guarantee made Layton leery of the patch, but Jan Kara noted that there never was a guarantee, even under the old behavior prior to `errseq_t`. 

A writeback error may not actually be reported by `fsync()`, however; other calls, such as `close()`, could return it. There was some discussion about whether getting an error return from a `close()` call actually makes sense. Dave Chinner said that a write failure that is not noticed until well after the write has completed (during writeback, for example) could be reported when the file is closed. 

But Layton mentioned Neil Brown's assertion that returning errors on close is not reasonable. For one thing, it is quite common for applications to ignore the return code from `close()`. But it is documented that the `close()` call can return errors, so some users will be dependent on _that_ behavior, Chinner said. 

Even with Wilcox's patch, there is still the problem of inodes that get evicted from memory. The bug has always been there but, since error reporting is being scrutinized and fixed, it is worth eliminating that problem. There was some talk of making the error persistent on disk, but Eric Sandeen suggested simply refusing to evict inodes that have errors associated with them. That should work well, and remove that lingering problem for PostgreSQL (and others) unless there is major memory pressure, Layton said. But Chinner does not think hanging on to a few extra inodes is going to affect the out-of-memory handling in any significant way. 

Layton mentioned that there is another problem: [`syncfs()`](<http://man7.org/linux/man-pages/man2/sync.2.html>) is "really broken" in its error reporting. He plans to fix that, probably by using another `errseq_t` in the superblock, since reporting from `syncfs()` requires a separate cursor on the error state. 

As the session was wrapping up, Chinner asked about tests for the changes. Layton said that he had some to add to xfstests, but there is a need to create some specifically for the PostgreSQL `fsync()` problem. That will allow some assurance that the problem has really been solved and that it doesn't regress down the road.
