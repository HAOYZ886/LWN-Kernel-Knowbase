---
title: "A reference-count tracking infrastructure"
url: https://lwn.net/Articles/877603/
date: "December 6, 2021"
category: Development tools; Reference counting
author: "By Jonathan Corbet December 6, 2021"
---

By **Jonathan Corbet**  
December 6, 2021

Reference counts are a commonly used mechanism for tracking the life cycle of objects in a computing system. As long as every user of an object correctly maintains its references by incrementing and decrementing the reference count, that object will persist for as long as it is needed and will be properly destroyed once the last user is done. The "correctly" in that sentence is important, though; things do not work as well in the presence of reference-counting errors. Networking developer Eric Dumazet is working on [a reference-count tracking system](<https://lwn.net/ml/netdev/20211205042217.982127-1-eric.dumazet@gmail.com/>) that could prove useful for finding these errors in the networking subsystem and, someday, throughout the kernel. 

Bugs in reference-count manipulation can be hard to find because the references themselves are anonymous. It may become clear, for example, that some user of an object has failed to release a reference before forgetting about that object, but there is generally no way to know _which_ user has done this. So the kernel ends up with an unused object that cannot be released, but there is no way to know where the reference-counting mechanism failed, or even which reference was lost. If there were a way to determine which of (say) several dozen references to an object was leaked, the task of finding the erroneous release path would be made considerably easier. 

Dumazet's patch set addresses this problem by creating a tracking mechanism that essentially surrounds the use of reference counts. The first step in using reference-count tracking is to add the tracker itself, in the form of a `ref_tracker_dir` structure: 

```
struct ref_tracker_dir {
        #ifdef CONFIG_REF_TRACKER
    	spinlock_t		lock;
    	unsigned int		quarantine_avail;
    	refcount_t		untracked;
    	struct list_head	list; /* List of active trackers */
    	struct list_head	quarantine; /* List of dead trackers */
        #endif
        };
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

This structure does not contain the reference count itself; instead, it is added to the object that contains the reference count of interest. The first use is in the massive [`net_device` structure](<https://elixir.bootlin.com/linux/v5.16-rc3/source/include/linux/netdevice.h#L1690>) that, as one might expect, is used to manage network devices. These structures are referenced in many places in the networking subsystem and are prone to reference-counting bugs; references are taken and released so often that a bespoke, per-CPU reference-counting mechanism is used. One leaked reference is enough to prevent the removal of a network device from the system; that, in turn, can block the clean-up of containers or virtual machines. To reduce the pain of finding these bugs, Dumazet's patch set starts by adding a `ref_tracker_dir` structure to `struct net_device`. This structure is initialized (via a required call to `ref_tracker_dir_init()`) with the lists empty and `quarantine_avail` (to be discussed below) set to a caller-supplied value. 

For the reference-count tracking mechanism to work properly, every code path that takes or releases a reference must also tell the tracker about that action. That is relatively easily done in settings where reference-count manipulation is encapsulated within some other function; otherwise a fair number of code changes may be required. Whenever the code takes a tracked reference to an object, it should make a call to: 

```
int ref_tracker_alloc(struct ref_tracker_dir *dir, struct ref_tracker **trackerp,
    			  gfp_t gfp_flags);
```

The `dir` argument points to the `ref_tracker_dir` structure added to the reference-counted object. This function will allocate a `ref_tracker` structure to track this specific reference, using the given `gfp_flags`, and storing its address in `*trackerp`. The return value is zero or an error code (mostly likely `-ENOMEM` if it is not possible to allocate the `ref_tracker` structure). 

`ref_tracker_alloc()` will add the new structure to `list` in the `ref_tracker_dir` structure, thus noting that a reference has been taken. To be useful, though, this tracker must somehow make a note of _where_ this specific reference was taken. The name of the function taking the reference would be useful, but often the real bug is several layers up the call stack, so a bit more information is needed to identify the real source of a problem. The tracking mechanism uses a kernel debugging feature called "stackdepot", which is able to generate and save full stack traces; stackdepot is rigorously undocumented, but the source is in [`lib/stackdepot.c`](<https://elixir.bootlin.com/linux/v5.16-rc3/source/lib/stackdepot.c>). By storing the full stack trace, the tracking mechanism gains the ability to shed light on the real source of reference-counting problems. 

When a reference is released, a call must be made to: 

```
int ref_tracker_free(struct ref_tracker_dir *dir, struct ref_tracker **trackerp);
```

This function will do a number of things, starting by removing the tracker pointed to (indirectly) by `trackerp` from `list`. The tracker is marked internally as having been released, but that structure is not immediately freed; instead, it is added to the `quarantine` list and the current stack trace is stored within it. This is done to catch double-free bugs; if `ref_tracker_free()` is called on a `ref_tracker` structure that is marked as already having been released, an immediate report is generated showing the stack traces associated with the allocation event and both free events. 

With busy objects, the `quarantine` list could easily grow to great length, so the above-mentioned `quarantine_avail` count is used to limit its length. Whenever a tracker is added to `quarantine`, the count is checked; if `quarantine_avail` is zero, the oldest tracker on the quarantine list will be freed, otherwise the count will be decremented. The initial value of `quarantine_avail` is specified at `ref_tracker_dir` initialization time; for network devices, it can grow to 128 entries. 

Double-free bugs can be caught immediately, but leaked references can only be detected when the time comes to free the reference-counted object. When the kernel is instructed to remove a network device, it will wait until the reference count on that device reaches zero; at this point, a leaked reference makes itself known by blocking the operation until the system is restarted. By this time, of course, the information about which reference has been leaked is long gone — at least, until the reference-count tracking mechanism comes into play. Any references taken to the object that were never released will still have active `ref_tracker` structures sitting around. A simple call to `ref_tracker_dir_print()` will cause the stack traces associated with any leaked references to be printed to the system log. 

The advantage of this mechanism is clear: it should be able to shine a light on a reference-counting bugs, easing the process of debugging those errors. On the other hand, it is a fairly heavyweight mechanism that is not suitable for production use. The implementation also requires that reference-count tracking be separately plumbed into every subsystem that wants to use it, and the changes may not be small; 21 patches out of Dumazet's 23-part series are dedicated to instrumenting `struct net_device`. A debugging mechanism transparently built directly into the `refcount_t` type would potentially be more applicable and less intrusive, but it wouldn't work with the one-off mechanism used with network devices. 

So this work is not, yet, a general-purpose reference-count debugging tool, but it is a significant step in that direction. The pieces are there waiting for a sufficiently motivated developer to turn them into something more general. Meanwhile, what's there should help to reduce the number of reference-counting bugs in the networking code, which is a good start.
