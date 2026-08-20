---
title: Retrying revoke()
url: https://lwn.net/Articles/546537/
date: "April 9, 2013"
category: revoke
author: "By Jonathan Corbet April 9, 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 9, 2013

The `revoke()` system call has one conceptually simple job: close any open file descriptors for the pathname given as its argument and prevent any further access via those descriptors. The classic use case is to ensure that no evil programs are holding a terminal or console device open before allowing logins there, but others exist as well. For example, a functioning `revoke()` implementation could be used within the kernel to cleanly disconnect any file descriptors referring to a device that has been removed from the system; filesystems like `/proc` could also use it to cleanly remove no-longer-needed virtual files. 

There is only one problem: Linux does not support `revoke()`, and every attempt to add it over the years has ended in failure. The functionality behind `revoke()` turns out to be quite difficult to implement in a safe way. The latest [attempt](<https://lwn.net/Articles/546540/>) at a `revoke()` implementation may well come to a similar conclusion; there is not even a proof-of-concept patch to evaluate, after all. But, since the developer behind it is Al Viro, one assumes that its chances of success are mildly better than average. 

Not every file or device will support `revoke()`; in some cases, it may still prove too hard to do properly. With Al's proposal, in cases where revocation _is_ supported, there would be a new structure associated with the relevant device (or other) structure: 

```
struct revokable {
    	atomic_t in_use;		// number of threads in methods,
    	spinlock_t lock;
    	hlist_head list;
    	struct completion *c;
    	void (*kick)(struct revokable *);
        };
```

The `in_use` field is charged with tracking how many threads are actively executing in the `file_operations` methods associated with this object. Performing this tracking would require changing every method call site throughout the kernel to call a couple of helper functions and check for a revoked file. So a call that currently looks like: 

```
ret = file->f_op->read(...);
```

Would be turned into something like: 

```
if (start_using(file)) {
    	ret = file->f_op->read(...);
    	stop_using(file);
        } else {
    	ret = -EIO;  /* File revoked */
        }
```

The `start_using()` and `stop_using()` helper functions increment and decrement the `in_use` counter. If that counter is negative, though, access is being revoked and `start_using()` will return `false`; in such cases, the `file_operations` method should not be called and an appropriate error code should be returned. Naturally, the details of these helper functions are a bit more complex than this; see Al's posting for a more complete story. As Al notes, there are quite a few call sites for `file_operations` methods in the kernel, so this particular change would be relatively intrusive. 

The purpose of the `kick()` callback is to instruct the object's driver that access is being revoked and any outstanding I/O operations should be brought to an end. Processes waiting on I/O should return with an error code and the I/O canceled. After the `kick()` call, the number of threads running within the object's `file_operations` should quickly drop to zero. 

When `open()` is called on an object that supports revocation, the associated `file` structure will gain a pointer to a structure like: 

```
struct revoke {
    	struct file *file;
    	struct revokable *revokable;
    	struct hlist_node list;
    	bool closing;
    	struct completion *c;
        };
```

The `list` field is used to track all open files associated with a given revocable object. As the last step in an `open()` implementation, the `make_revokable()` helper will be called to allocate the `revoke` structure and attach it to the list in the object's `revokable` structure. 

With this infrastructure in place, an implementation of `revoke()` becomes possible. The steps, roughly, are these: 

  1. Mark the object as being revoked by subtracting a large number from its `in_use` counter, turning that counter negative. That will prevent any further calls to the object's `file_operations` methods. 

  2. If `in_use` indicates that threads are currently running in the object's `file_operations`, call `kick()` to encourage them all to finish and wait until they all complete. 

  3. For each open file, call the `release()` method to close that file, and remove the file from the list. 
At the end of this process, there should be no open files for the given object and no threads will be running in any file operations associated with that object. The latter point is important; a robust and secure `revoke()` implementation is possible only if the kernel can be sure that all previous references to the revoked object are truly gone. Once that has happened, it should then be possible to free any associated resources or allow new processes to open the object. 

There is, of course, one other thorny little problem: what do to about processes that have used `mmap()` to map the object into their address space. One possibility is to forcibly unmap the memory, tearing down the associated page tables and marking the virtual memory area (VMA) structure accordingly; the process would then most likely receive a `SIGSEGV` signal if it attempted to access that address space. That approach is secure, but also risks causing programs to crash unexpectedly. In cases where device memory has been mapped, a better solution might be to just cause all accesses to return `0xff` (extended out to the correct width for the specific access). Proper handling of `mmap()` in this situation is an open question, and one apparently without precedent in the current implementations of `revoke()` in other systems — `revoke()` on BSD systems works only on devices without mapped memory. 

There is a fair gap between an RFC posting with a clever idea and an actual, working implementation; it may well be that this approach to `revoke()` will, like its predecessors, run aground in the real world. But the lack of a working `revoke()` has been seen as a shortcoming in Linux for many years; it would be nice to finally get this functionality into place. So, just maybe, things will work out this time around.
