---
title: Epoll evolving
url: https://lwn.net/Articles/633422/
date: "February 16, 2015"
category: Epoll
author: "By Jonathan Corbet February 16, 2015"
---

By **Jonathan Corbet**  
February 16, 2015

Epoll is a set of Linux-specific system calls intended to provide fast polling for large numbers of file descriptors. The API has been in use since it was merged during the 2.5 development series but, like many interfaces, there is always room for improvement. There are currently two patch sets making the rounds that would add functionality to this set of system calls. 

#### An epoll overview

Epoll is designed to function somewhat like `select()` or `poll()`, but with more options and with higher performance when large numbers of file descriptors are in use. Each call to `select()` or `poll()` can involve an entirely new set of file descriptors, so the kernel must go through the process of validating each one, checking for I/O readiness, and adding the polling process to the appropriate wait queue. But the actual list of file descriptors tends not to change much between calls, so much of that work is unnecessary duplicated effort. The epoll calls get around this problem by separating that setup work from the act of waiting for a file descriptor to become ready. 

A process using this API must begin by creating a special file descriptor to use with polling; that is done with a call to one of: 

```
#include <sys/epoll.h>
    
        int epoll_create(int size);
        int epoll_create1(int flags);
```

Either call will return a file descriptor to be used with the remaining epoll functions. The `size` parameter to `epoll_create()` is no longer used; the `flags` argument added for `epoll_create1()` can be used to set the close-on-exec flag for the resulting file descriptor. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The next step is to add all of the file descriptors that are to be monitored, using: 

```
int epoll_ctl(int efd, int op, int fd, struct epoll_event *event);
```

If `op` is `EPOLL_CTL_ADD`, the given `fd` will be added to the set. The `event` parameter is used to describe which events will be polled for; see [the man page](<http://man7.org/linux/man-pages/man2/epoll_ctl.2.html>) for details. `epoll_ctl()` can also be used to remove file descriptors or to modify how the polling is done. 

Finally, actually waiting for one of the file descriptors in the set to become ready is done with: 

```
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
        int epoll_pwait(int epfd, struct epoll_event *events, int maxevents, int timeout,
                        const sigset_t *sigmask);
```

The events that cause `epoll_wait()` to return (up to `maxevents`) will be stored in the `events` array; the `timeout` parameter is specified in milliseconds. The `epoll_pwait()` version also allows the specification of a set of signals to be blocked (or unblocked) during the call. Again, see [the man page](<http://man7.org/linux/man-pages/man2/epoll_wait.2.html>) for details. 

#### epoll_ctl_batch() and epoll_pwait1()

[This patch set from Fam Zheng](<https://lwn.net/Articles/633195/>) introduces two new system calls to the epoll family. The first of these addresses a performance problem that comes up in situations where it is often necessary to make changes to the file descriptors in the epoll set. A call to `epoll_ctl()` can only add, modify, or delete a single file descriptor; if a lot of descriptors need to be changed, there will need to be a lot of `epoll_ctl()` calls to get that work done. The proposed `epoll_ctl_batch()` system call gets around that problem by processing multiple file descriptors in a single call: 

```
int epoll_ctl_batch(int epfd, int flags, int ncmds, struct epoll_ctl_cmd *cmds);
```

The `cmds` structure essentially duplicates all of the arguments that would have been passed to an `epoll_ctl()` call. By passing an array of these structures, a program can perform operations on multiple file descriptors with a single system call. 

Fam's other change is to add a new system call to perform the actual polling: 

```
struct epoll_wait_params {
    	int clockid;
    	struct timespec timeout;
    	sigset_t *sigmask;
    	size_t sigsetsize;
        };
    
        int epoll_pwait1(int epfd, int flags,
                         struct epoll_event *events, int maxevents,
                         struct epoll_wait_params *params);
```

This version of `epoll_wait()` adds a new `flags` parameter, but does not define any flag values, so `flags` must be zero. The parameters shuffled off into the `params` structure are mainly intended to give the application more control over timeout processing. The millisecond-resolution timeout understood by `epoll_wait()` has proved to be too coarse for a number of use cases. The new system call defines a timeout with nanosecond resolution, getting around that problem. 

#### Better multi-threaded behavior

Jason Baron (of Akamai) has run into a different problem that only comes up with a relatively uncommon usage pattern. Normally, there will only be one process using a polling function to monitor a given file descriptor. But, in Jason's use case, there can be multiple threads, each of which is using epoll to track the same set of file descriptors. In this setting, an event on any given file descriptor will cause _all_ of the waiting processes to wake up, even though only one of them will be able to actually handle the event. This thundering-herd problem is something that Jason would like to avoid. 

His response is [this patch](<https://lwn.net/Articles/632590/>) adding a couple of new flags to be attached to file descriptors via `epoll_ctl()`. The first of these, `EPOLLEXCLUSIVE`, requests that only one process be woken up to handle an event on the associated file descriptor. Internally, the change is a simple matter of using `add_wait_queue_exclusive()` instead of `add_wait_queue()` when setting up the polling. Obviously, all processes polling the same file descriptor would have to use the exclusive mode to get down to a single wakeup per event. 

That change did not entirely solve Jason's problem, though, in that it ended up with the same process waking up in response to each event. Since one of the reasons for epoll's existence is to allow processes to be left on all of the file-descriptor-specific wait queues between calls to `epoll_wait()`, the process that is at the head of any given queue will remain there. It will thus be the one to receive all of the exclusive wakeups. But the whole point of having multiple processes polling the same file descriptor is to spread the work around; waking the same process every time thwarts that objective. To deal with that, Jason added another flag, called `EPOLLROUNDROBIN`, that causes the kernel to work through each of the polling processes in turn. 

Support for this mode was added to the scheduler in the form of a new wait queue function: 

```
void add_wait_queue_rr(wait_queue_head_t *q, wait_queue_t *wait);
```

When waiting is done using `add_wait_queue_rr()`, only a single process will wake, just like with `add_wait_queue_exclusive()`. But, in addition, the process at the head of the wait queue will be moved to the tail, so it will not see another wakeup until all of the other processes have had their turn. 

Jason's patch posting includes results from a little benchmark program showing a nearly 50% reduction in execution time when the exclusive mode is used. When there are a lot of wakeups happening (as can be the case for many network-oriented workloads), the extra overhead caused by thundering herds can be crippling. 

Both patches have received a fair number of review comments, and Fam's patches in particular have evolved quite a bit since [the first posting](<https://lwn.net/Articles/630097/>) in January. Your editor's unscientific impression is that API-related patches are getting more attention than they once did. That, of course, would be a good thing; APIs live forever, so it's better to get them right before shipping them to users and being committed to supporting them forever. These patches seem close to being ready, though; they might make an appearance in the next merge window.
