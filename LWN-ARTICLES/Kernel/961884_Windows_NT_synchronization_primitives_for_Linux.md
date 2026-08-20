---
title: Windows NT synchronization primitives for Linux
url: https://lwn.net/Articles/961884/
date: "February 16, 2024"
category: "Locking mechanisms; Releases-6.10"
author: "By Jonathan Corbet February 16, 2024"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
February 16, 2024

The [futex](<https://man7.org/linux/man-pages/man2/futex.2.html>) mechanism provided by the kernel allows for the creation of efficient and flexible locking primitives in user space. Futexes work well for many applications, but not all. One of the exceptions, it seems, is that perennially difficult-to-support use case: Windows games. With [this patch series](<https://lwn.net/ml/linux-kernel/20240214233645.9273-1-zfigura@codeweavers.com/>), Elizabeth Figura seeks to provide the sort of locking that those games need, by way of a special-purpose virtual device. 

The performance of a futex can be hard to beat when it is used as intended; in the uncontended case, there is no need for a system call at all to acquire one. Surprisingly, though, the Windows NT locking primitives were not designed with the objective of being easy to implement efficiently with futexes; as a result, there are certain operations supported by Windows that are not straightforward to implement on Linux. At the top of the list is operations requiring the simultaneous acquisition of multiple locks. 

Applications written for Unix-like systems normally do not suffer from the lack of Windows-style locking primitives, but Windows applications that have been made to run on Linux often will. Until now, these applications have been supported in [Wine](<https://www.winehq.org/>) by creating a special process to arbitrate access to locks. That solution can work, but it adds an interprocess-communication overhead to every locking operation, which hurts performance. The new device takes the place of that process, handling locking in the kernel without the communication overhead. 

To use this feature, a process opens the new special file `/dev/ntsync`. Every open of that file creates a new instance that is distinct from all of the others, so the intended use case is a single process that shares an instance across multiple threads. Each instance provides a whole set of [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) operations (all described on [this patch](<https://lwn.net/ml/linux-kernel/20240131021356.10322-30-zfigura@codeweavers.com/>)). The first step to use those operations will be to create the locks to be managed by the device; they come in three flavors: 

  * A **mutex** is similar to the kernel equivalent; it is a lock that can be held by a single owner at a time. Locking calls can nest, though: once a thread has acquired a mutex it can do so again any number of times. Once all of the acquisition calls have been matched with release calls, the mutex is freed. 
  * A **semaphore** is a counter, as one would expect. Every acquisition decrements the counter by one; as long as the counter is nonzero, the semaphore remains available. 
  * An **event** is a condition variable; it has a boolean value, and threads can wait until it becomes true. If the event is marked for auto-reset, it will be reset to false as soon as a wait is satisfied, meaning that only one thread will see the event become true. Otherwise, an event, once set to true, stays that way until explicitly reset. 

The `NTSYNC_IOC_CREATE_MUTEX`, `NTSYNC_IOC_CREATE_SEM`, and `NTSYNC_IOC_CREATE_EVENT` `ioctl()` calls can be used to create a mutex, semaphore, or event, respectively. On success, each of these operations returns a file descriptor that can be used to operate on the created object. The API is a bit different than one might expect, in that the file descriptor is not the return value from `ioctl()`; instead, it is stored in a structure passed by user space. 

For example, to create a mutex, a thread starts with this structure: 

```
struct ntsync_mutex_args {
       	__u32 mutex;
       	__u32 owner;
       	__u32 count;
       };
```

On entry to the `NTSYNC_IOC_CREATE_MUTEX` call, the value of `mutex` is ignored. The `owner` field is set to the (application-defined) ID of the initial owner of the mutex, while `count` is set to the number of times the mutex has been acquired by that owner. To create a mutex that is not yet owned by anybody, both of those fields will simply be set to zero. On a successful return, the file descriptor corresponding to this mutex will be stored in the `mutex` field. 

A number of operations are provided for manipulating these objects. For mutexes, `NTSYNC_IOC_READ_MUTEX` will return the current state of a mutex, while `NTSYNC_IOC_MUTEX_UNLOCK` will unlock a (currently locked) mutex. A slightly strange one is `NTSYNC_IOC_KILL_OWNER`, which doesn't actually kill anything; it takes a thread ID as an argument and, if that ID is the owner of the mutex, that mutex will be freed and marked as "abandoned". The next attempt to acquire the mutex will return an error status of `EOWNERDEAD`, but the acquisition will have succeeded anyway. 

For semaphores, `NTSYNC_IOC_READ_SEM` will read the current state, and `NTSYNC_IOC_SEM_POST` will add a given amount to the semaphore's count (perhaps releasing the semaphore). Events can be queried with `NTSYNC_IOC_READ_EVENT` and modified with `NTSYNC_IOC_SET_EVENT`, `NTSYNC_IOC_RESET_EVENT`, and `NTSYNC_IOC_PULSE_EVENT`. That last operation acts like an instantaneous set and reset of the event, allowing one or more waiting threads to proceed but never causing the event to appear to be set. The "pulse" operation is one of those that is hard to implement with futexes. 

To actually acquire a mutex or semaphore involves calling either `NTSYNC_IOC_WAIT_ANY` (which will return as soon as it is able to acquire any one of a list of mutexes and semaphores or one of the indicated events is set) or `NTSYNC_IOC_WAIT_ALL`, which will only return when it is able to atomically acquire _all_ of the indicated resources. The latter operation will make an attempt whenever one of the resources is freed, but will only succeed if all of them happen to be available. It will not hold a partial set of resources while waiting for the rest, so it could be subject to starvation if the resources are heavily contended. Both wait operations include an optional timeout. 

The motivation behind this work becomes clear after a look at the benchmark results provided in the patch cover letter: 

> The gain in performance varies wildly depending on the application in question and the user's hardware. For some games NT synchronization is not a bottleneck and no change can be observed, but for others frame rate improvements of 50 to 150 percent are not atypical. 

The question that has not been directly answered in the cover letter is whether the futex API could have been enhanced to provide the needed functionality without introducing an entirely new API. It would seem (though your editor, needless to say, has not tried to implement it) that the "pulse event" functionality would be relatively straightforward to add. Some aspects of the multi-resource wait operations were provided by the addition of [`futex_waitv()`](<https://lwn.net/Articles/866112/>) to the 5.16 kernel, but more work would clearly have to be done. It may well be that adding a standalone virtual device for this niche functionality is easier and less intrusive than trying to coerce futexes into doing the job. 

The comments on [the first version](<https://lwn.net/ml/linux-kernel/20240124004028.16826-1-zfigura@codeweavers.com/>) of the patch set were focused on the details of the API rather than whether a separate device was needed; they resulted in a number of changes leading to the API described here. Subsequent versions, the last of which was posted on February 14, have received relatively few comments so far. So, perhaps, the community is happy with this proposal in its current form, and Linux gamers can look forward to a 131% faster Lara Croft in the near future.
