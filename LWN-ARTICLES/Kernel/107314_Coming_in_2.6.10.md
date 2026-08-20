---
title: Coming in 2.6.10
url: https://lwn.net/Articles/107314/
date: "October 20, 2004"
category: Block layer; Circular buffers; Events reporting; IO scheduler; kfifo
author: 
---

A large number of patches have already been merged and will show up in the first 2.6.10 prepatch. Some of those have been covered on this page before, but others have not. As a way of catching up with current events, we'll take a quick look at a few of these patches. 

### CFQ v2

The completely fair queueing (CFQ) I/O scheduler endeavors to get good performance from block devices while dividing the available bandwidth equally between the processes contending for each device. 2.6.10 will contain a major rework of the CFQ scheduler, called "CFQ v2." Some of the changes in this version are: 

  * Process I/O context information is maintained for the lifetime of each process, rather than just for the periods when the process has outstanding I/O. This change fixes some starvation scenarios which came up with CFQ v1. 

  * Grouping of processes can be done by user ID, group ID, thread group, or process group; the policy in force can be changed at runtime. 

  * Request ordering is more strictly enforced as a way of limiting the maximum latency experienced by any given request. 

  * Small backward seeks are occasionally allowed if they look like they will improve responsiveness. 

The code is also more heavily commented; author Jens Axboe says that was done to increase its AAF - "akpm acceptance factor." AKPM is Andrew Morton, who has been known to complain about insufficiently commented kernel submissions. 

### Simple circular buffers

Circular buffers are a common data structure in the kernel, but there has never been a generic implementation available for use. Stelian Pop decided to change that; he was almost certainly surprised, however, by the large number of iterations it took to respond to all the comments he got. In the end, this effort showed the value of having a single, generic implementation in the kernel. Even a data structure as simple as a circular buffer can be tricky to implement correctly; it makes no sense for every developer to go through that process each time a new one is needed. With a single, well-reviewed implementation, the chances of it being truly correct are much better. 

A circular buffer is represented by `struct kfifo`, defined in `<linux/kfifo.h>`. A staticly-allocated buffer can be initialized with `kfifo_init()`, or allocation and initialization can be performed together with `kfifo_alloc()`: 

```
struct kfifo *kfifo_init(unsigned char *buffer, unsigned int size,
                             int gfp_mask, spinlock_t *lock);
    struct kfifo *kfifo_alloc(unsigned int size, int gfp_mask,
                              spinlock_t *lock);
```

Either way, `size` is the desired size of the buffer (in bytes, must be a power of two), `gfp_mask` is a set of `GFP_` flags controlling how memory allocations will be performed, and `lock` is a spinlock which will be used to serialize access to the data structure. 

The functions for moving data into and out of the buffer are: 

```
unsigned int kfifo_put(struct kfifo *fifo, unsigned char *buffer, 
                           unsigned int len);
    unsigned int kfifo_get(struct kfifo *fifo, unsigned char *buffer, 
                           unsigned int len);
```

These functions move at most `len` bytes between the structure and `buffer`; the actual number of bytes transferred is returned. The number of bytes currently stored in a circular buffer can be obtained by passing it to `kfifo_len()`, and a buffer may be flushed by passing it to `kfifo_reset()`. A dynamically-allocated buffer may be returned to the system with `kfifo_free()`; there does not seem to be a way to free memory from staticly-allocated buffers. 

### Kernel events

The kernel events notification mechanism has been covered here a couple of times. This code provides a way for user-space processes to learn about important events by way of a netlink socket. The final form of the event generation interface (for now) is: 

```
int kobject_uevent(struct kobject *kobj, enum kobject_action action,
                           struct attribute *attr);
```

The kobject describes where the interesting event happened. For the one explicit use currently in the kernel (filesystem mount and unmount events), the kobject corresponds to the disk partition involved. `action` is a small set of possible events; it is currently one of `KOBJ_ADD`, `KOBJ_REMOVE`, `KOBJ_CHANGE`, `KOBJ_MOUNT`, and `KOBJ_UMOUNT`. The "add" and "remove" actions are generated along with hotplug events; "change" describes attribute value changes, and "mount" and "unmount" are for filesystem events. The final parameter (`attr`) is an optional attribute of the given kobject which provides further information. 

The patches merged also modify how hotplug events are handled; such events now are reported in two ways: via the new events mechanism and through an invocation of `/sbin/hotplug`.
