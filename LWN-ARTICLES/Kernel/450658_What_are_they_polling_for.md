---
title: "What are they polling for?"
url: https://lwn.net/Articles/450658/
date: "July 7, 2011"
category: "Device drivers-Support APIs; poll"
author: "By Jonathan Corbet July 7, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 7, 2011

The `poll()`, `select()`, and `epoll_wait()` system calls all allow an application to ask the kernel whether I/O on any of a list of file descriptors would block and, optionally, to wait until one or more descriptors become ready for I/O. Internally, they are all implemented with the `poll()` method in the `file_operations` structure: 

```
unsigned int (*poll) (struct file *filp, struct poll_table_struct *pt);
```

This function returns a value indicating whether non-blocking I/O is currently possible; it is also expected to add a wait queue to the "poll table" (`pt`) passed in. If no file descriptors are ready for I/O, the calling process will block on all of the accumulated wait queues. 

`poll()` has long implemented an optimization: if an early `poll()` function indicates that I/O is possible, the kernel knows that it will not be blocking the calling process. So it stops accumulating wait queues; this state is indicated by passing a null pointer for `pt`. That all works well except in one case: what if a driver needs access to some of the information stored in the poll table? 

In particular, the driver might want to know whether the caller is interested in readiness for read or write access, or whether it is looking for exceptional events. For example, if the application wants to read from the descriptor, the driver may need to fire up some device machinery to make that possible. This situation has not come up very often, but it does tend to affect Video4Linux drivers. In response, Hans Verkuil has posted [a patch](<https://lwn.net/Articles/449984/>) slightly changing the way `poll()` works. 

With the patch, the poll table is never passed as null; instead, the "we will not be blocking" case is marked internally. So the set of events requested by the application is always available; Hans has provided a helper function to access that information: 

```
unsigned long poll_requested_events(const poll_table *p);
```

There has been little discussion of the patch; it doesn't seem like there is any real reason for it not to go in for 3.1.
