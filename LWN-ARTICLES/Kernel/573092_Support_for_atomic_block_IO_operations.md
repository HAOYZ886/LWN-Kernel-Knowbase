---
title: Support for atomic block I/O operations
url: https://lwn.net/Articles/573092/
date: "November 6, 2013"
category: "Atomic IO operations; Block layer-Atomic operations"
author: "By Jonathan Corbet November 6, 2013"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 6, 2013

Some newer storage devices have the ability to perform [atomic I/O operations](<https://lwn.net/Articles/552095/>). An atomic operation will either succeed or fail as a unit; if multiple blocks are to be written, they will all make it to persistent storage or none will. This feature has the potential to improve life considerably at the higher levels, but the kernel currently has no way to support it. 

Chris Mason's [atomic I/O patch set](<https://lwn.net/Articles/572766/>) aims to fix that situation. It allows a file to be opened with the `O_ATOMIC` and `O_DIRECT` flags (only direct I/O is supported) to request atomic semantics. Thereafter, every `write()` call will be executed atomically if the hardware supports it. This feature is, thus, quite easy to use from user space. 

Within the kernel, there is a new function available to block drivers: 

```
void blk_queue_set_atomic_write(struct request_queue *q,
        				    unsigned int segments);
```

This function tells the block layer that the device behind the given request queue can perform atomic operations up to the given number of `segments` (separate ranges of blocks on the storage medium). Thereafter, I/O requests may arrive with the `REQ_ATOMIC` flag set to request atomic execution. The block layer will ensure that the maximum segment count is not exceeded. 

One can imagine a number of uses for this functionality. A journaling filesystem could, for example, use it to write out the journal and the commit block together, knowing that said commit block will only be visible if everything else was successfully written. But, Chris says, the first target is MySQL: 

O_ATOMIC | O_DIRECT allows mysql and friends to disable double buffering. This cuts their write IO in half, making them roughly 2x more flash friendly. 

The patch set (which does not add support to any block drivers) is relatively small and simple, so it should have a relatively good chance of being merged in the very near future.
