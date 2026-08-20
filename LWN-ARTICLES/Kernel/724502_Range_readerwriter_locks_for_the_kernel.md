---
title: Range reader/writer locks for the kernel
url: https://lwn.net/Articles/724502/
date: "June 5, 2017"
category: "Memory management-mmap sem; Range locks"
author: "By Jonathan Corbet June 5, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 5, 2017

The kernel uses a variety of lock types internally, but they all share one feature in common: they are a simple either/or proposition. When a lock is obtained for a resource, the _entire_ resource is locked, even if exclusive access is only needed to a part of that resource. Many resources managed by the kernel are complex entities for which it may make sense to only lock a smaller part; files (consisting of a range of bytes) or a process's address space are examples of this type of resource. For years, kernel developers have talked about adding "range locks" — locks that would only apply to a portion of a given resource — as a way of increasing concurrency. Work has progressed in that area, and range locks may soon be added to the kernel's locking toolkit. 

Jan Kara posted [a range-locking mechanism](<https://lwn.net/Articles/535843/>) in 2013, but that work stalled and never made it into the mainline. More recently, Davidlohr Bueso has [picked up that work](<https://lwn.net/Articles/722741/>) and extended it. The result is a new form of reader/writer lock — a lock, in other words, that distinguishes between read-only and write access to a resource. Reader/writer locks can increase concurrency in settings where the protected resource is normally accessed by readers, since all readers can run simultaneously. Whenever a writer comes along, though, it must have exclusive access to the resource. Balancing access between readers and writers can be a tricky business where the wrong decisions can lead to starvation, unfairness, or poor concurrency. 

Since range locks only cover part of a resource, there can be many of them covering separate parts of the resource as a whole. The data structure that describes all of the known range locks, including those that are waiting for the needed range to become available, for a given resource is a "range lock tree", represented by `struct range_lock_tree`. This "tree" is the lock that protects the resource as a whole; it will be typically located in or near the relevant data structure where one would otherwise find a simpler lock. Thus, a range-locking implementation will tend to start with something like: 

```
#include <linux/range_lock.h>
    
        DEFINE_RANGE_LOCK_TREE(my_tree);
```

Given the `range_lock_tree` structure to protect the resource, a thread needing access to a portion of that resource will need to acquire a lock on the range of interest. A lock on a specific range (whether granted or not) is represented by `struct range_lock`. It is possible to declare and initialize a range lock statically with either of: 

```
DEFINE_RANGE_LOCK(my_lock, start, end);
        DEFINE_RANGE_LOCK_FULL(name);
```

The second variant above will describe a lock on the entire range. It is also possible to initialize a `range_lock` structure at run time with either of: 

```
void range_lock_init(struct range_lock *lock, unsigned long start,
        			 unsigned long end);
        void range_lock_init_full(struct range_lock *lock);
```

Actually acquiring a range lock requires calling one of a large set of primitives. In the simplest case, a call to `range_read_lock()` will acquire a read lock on the indicated range, blocking if necessary to wait for the range to become available: 

```
void range_read_lock(struct range_lock_tree *tree, struct range_lock *lock);
```

The lock for the entire resource is provided as `tree`, while `lock` describes the region that is to be locked. Like most sleeping lock primitives, `read_range_lock()` will go into a non-interruptible sleep if it must wait. That behavior can be changed by calling one of the other locking functions: 

```
int range_read_lock_interruptible(struct range_lock_tree *tree,
    				      struct range_lock *lock);
        int range_read_lock_killable(struct range_lock_tree *tree, struct range_lock *lock);
        int range_read_trylock(struct range_lock_tree *tree, struct range_lock *lock);
```

In any case, a read lock that has been granted must eventually be released with: 

```
void range_read_unlock(struct range_lock_tree *tree, struct range_lock *lock);
```

If, instead, the range must be written to, a write lock should be obtained with one of: 

```
void range_write_lock(struct range_lock_tree *tree, struct range_lock *lock);
        int range_write_lock_interruptible(struct range_lock_tree *tree,
    				       struct range_lock *lock);
        int range_write_lock_killable(struct range_lock_tree *tree, struct range_lock *lock);
        int range_write_trylock(struct range_lock_tree *tree, struct range_lock *lock);
```

A call to `range_write_unlock()` will release a write lock. It is also possible to turn a write lock into a read lock with: 

```
void range_downgrade_write(struct range_lock_tree *tree, struct range_lock *lock);
```

The implementation does not give any particular priority to either readers or writers. If a writer is waiting for a given range, a reader that arrives later requesting an intersecting range will wait behind the writer, even if other readers are active in that range at the time. The result is, possibly, less concurrency than might otherwise be possible, but this approach also ensures that writers will not be starved for access. 

This patch set has been through a few revisions and does not seem to be generating much more in the way of comments, so it might be about ready to go. The first user is the Lustre filesystem, which is already using a variant of Kara's range-lock implementation internally to control access to ranges of files. But there is [a potentially more interesting user](<https://lwn.net/Articles/723648/>) waiting on the wings: using range locks as a replacement for `mmap_sem`. 

The reader/writer semaphore known as `mmap_sem` is one of the most intractable contention points in the memory-management subsystem. It protects a process's memory map, including, to an extent, the page tables. Many performance-sensitive operations, such as handling page faults, must acquire `mmap_sem` with the result that, on many workloads, contention for `mmap_sem` is a significant performance bottleneck. Protecting a process's virtual address space would appear to be a good application for a range lock. Most of the time, a change to the address space does not affect the entire space; it is, instead, focused on a particular set of addresses. Using range locks would allow more operations on a given address space to proceed concurrently, reducing contention and improving performance. 

The patch set (posted by Laurent Dufour) does not yet achieve that goal; instead, the entire range is locked every time. Thus, with these patches, a range lock replaces `mmap_sem` without really changing how things work. Restricting the change in this way allows the developers to be sure that the switch to a range lock has not introduced any bugs of its own. Once confidence in that change exists, developers will be able to start reducing the ranges to what is actually needed. 

These changes will need to be made with care, especially since what is being protected by `mmap_sem` is not always clear. But, given enough development cycles, the `mmap_sem` bottleneck should slowly dissolve away, leaving us with a faster, more concurrent memory-management subsystem. Some improvements are worth waiting for.
