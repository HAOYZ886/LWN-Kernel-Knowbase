---
title: Big reader locks
url: https://lwn.net/Articles/378911/
date: "March 16, 2010"
category: "Filesystems-Virtual filesystem layer; Scalability"
author: "By Jonathan Corbet March 16, 2010"
---

By **Jonathan Corbet**  
March 16, 2010

Nick Piggin's [VFS scalability patches](<http://lwn.net/Articles/360199/>) have been a work in progress for some time - as is often the case for this sort of low-level, performance-oriented work. Recently, Nick has begun to break the patch set into smaller pieces, each of which solves one part of the problem and each of which can be considered independently. One of those pieces introduces an interesting new mutual exclusion mechanism called the [big reader lock](<http://lwn.net/Articles/378781/>), or "brlock." 

Readers of the patch can be forgiven for wondering what is going on; anything which combines tricky locking and 30-line preprocessor macros is going to raise eyebrows. But the core concept here is simple: a brlock tries to make read-only locking as fast as possible through the creation of a per-CPU array of spinlocks. Whenever a CPU needs to acquire the lock for read-only access, it takes its own dedicated lock. So read-locking is entirely CPU-local, involving no cache line bouncing. Since contention for a per-CPU spinlock should really be zero, this lock will be fast. 

Life gets a little uglier when the lock must be acquired for write access. In short: the unlucky CPU must go through the entire array, acquiring every CPU's spinlock. So, on a 64-processor system, 64 locks must be acquired. That will not be fast, even if none of the locks are contended. So this kind of lock should be used rarely, and only in cases where read-only use predominates by a large margin. 

One such case - the target for this new lock - is `vfsmount_lock`, which is required (for read access) in pathname lookup operations. Lookups are frequent events, and are clearly performance-critical. On the other hand, write access is only needed when filesystems are being mounted or unmounted - a much rarer occurrence. So a brlock is a good fit here, and one small piece (out of many) of the VFS scalability puzzle has been put into place.
