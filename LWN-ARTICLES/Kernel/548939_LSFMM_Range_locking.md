---
title: "LSFMM: Range locking"
url: https://lwn.net/Articles/548939/
date: "May 1, 2013"
category: Filesystems
author: "By Jake Edge May 1, 2013 LSFMM Summit 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
May 1, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

At the 2013 LSFMM Summit, Jan Kara led a discussion of his [proposal for range locking](<https://lwn.net/Articles/535843/>). It is an in-kernel mechanism that would allow locking ranges of pages within a file, rather than locking the entire file. Currently, there are several synchronization primitives used to serialize page cache access, but some of the operations are not serialized, which can lead to races. 

Range locking has a number of advantages, including fixing a race condition when punching holes in files that can result in filesystem corruption. Nothing prevents a page fault from happening when punching a hole, so there is a race with `mmap()`. There is also a need for a synchronizing mechanism when truncating the page cache (e.g. for direct I/O). A range lock will prevent the creation of pages within the range that is covered by the lock. 

For filesystems that are changed to take advantage of it (or changed to do so for certain cases), range locks can replace the `i_mutex` (inode mutex). The patches to implement range locking are around 200 lines of code, Kara said. It is not a complex piece of code, either, just using a pair of mutexes, he said. The good news is that the way it is implemented, the overhead is minimal. 

The reaction was generally favorable, though more testing is probably required.
