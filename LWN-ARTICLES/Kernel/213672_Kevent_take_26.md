---
title: Kevent take 26
url: https://lwn.net/Articles/213672/
date: "December 12, 2006"
category: Events reporting; Kevent
author: 
---

Some patches make it into the kernel in something very close to their original form. Others have to go through a few changes first. The all-time record for development iterations may be held by devfs; Richard Gooch had just released [the 157th revision](<http://lwn.net/2000/0217/a/devfs-157.html>) when this ill-fated subsystem was merged for 2.3.46. On that scale, Evgeniy Polyakov is just getting started with [kevent take 26](<http://lwn.net/Articles/213631/>); even so, the process must be starting to seem like a long one. 

In this case, however, the long process can be seen as evidence that the system is working as it should. The kevent subsystem is a major addition to the Linux system call API. Once it goes in, it will have to be supported forever (to a finite-precision arithmetic approximation, at least). Adding a kevent interface with warts, or which does not provide the best performance possible, would be a serious mistake. Nobody wants to be faced with designing and implementing a new event interface in a few years while supporting the old one indefinitely. So it makes sense to go slowly and make sure that things have been thought out well. 

The number of people posting comments on the kevent patches has been relatively small; for whatever reason, many normally vocal developers do not seem to have much to say on this new API. Fortunately, Ulrich Drepper (the glibc maintainer) has taken a strong interest in this interface and has pushed hard for the changes he thought were necessary. One gets the sense the Ulrich and Evgeniy have gotten a little tired of each other over the last month or so. But, to their credit, they have stuck to the task. As of this writing, Ulrich has not commented on the version of the API implemented in the "take 26" patch set. It does, however, clearly reflect some of the things he has been asking for. 

While Evgeniy has been concerned with getting events out of the kernel, Ulrich has been worried about performance and robustness. So he wanted ways for multi-threaded programs to cancel threads at any time without losing track of which events have been processed. Whenever possible, he would like to be able to process events without involving the kernel at all. And he has pushed strongly for timeout values to be represented in an absolute format. Evgeniy has (a bit grudgingly, at times) addressed most of these wishes. 

It is still possible to get a kevent file descriptor by opening `/dev/kevent`, though that is no longer the only way. The `kevent_ctl()` system call is still used for the management of events: 

```
int kevent_ctl(int fd, unsigned int cmd, unsigned int num, 
                       struct ukevent *arg);
```

With `kevent_ctl()`, an application can add requests for events, remove them, or modify them in place. There is a new `KEVENT_CTL_READY` operation which can be used to mark specific events as being "ready" and cause the kernel to wake up one or more processes waiting for events. 

The synchronous interface has been changed slightly: 

```
int kevent_get_events(int ctl_fd, unsigned int min_nr, 
                              unsigned int max_nr, struct timespec timeout, 
    			  struct ukevent *buf, unsigned flags);
```

The difference is that the timeout value now is a `struct timespec`. That value is still interpreted as a relative timeout, however, unless `flags` contains `KEVENT_FLAGS_ABSTIME`. In the latter case, `timeout` is an absolute time, and the code will print a warning to the effect that Evgeniy was wrong in believing that nobody would ever want to use absolute times. 

It is expected, however, that performance-aware applications will use the user-space ring buffer rather than the synchronous interface. That ring buffer is still set up with `kevent_init()`: 

```
int kevent_init(struct kevent_ring *ring, unsigned int ring_size,
                        unsigned int flags);
```

The file descriptor argument has been removed from this system call; instead, `kevent_init()` opens a new file descriptor and passes it back as its return value. Thus, there is no separate need to open `/dev/kevent`. 

The `kevent_ring` structure has changed a bit since it was last discussed on this page: 

```
struct kevent_ring
        {
            unsigned int ring_kidx, ring_over;
       	struct ukevent event[0];
        };
```

The new `ring_over` value counts the number of times that the index into the ring has wrapped around. This parameter is used to ensure that the kernel and the application have the same understanding of the state of the ring buffer before allowing the application to mark events as being consumed. 

Waiting for events to arrive in the ring is done with `kevent_wait()`, which now looks like this: 

```
int kevent_wait(int ctl_fd, unsigned int num, unsigned int old_uidx, 
     	            struct timespec timeout, unsigned int flags);
```

Here, too, the timeout value is a `struct timespec`, and, once again, absolute timeouts must be marked with the `KEVENT_FLAGS_ABSTIME` flag. This call will wait until at least one event is ready, then copy up to `num` events into the ring buffer. The `old_uidx` is the index of the last event that the calling application knows about; if more events are added between when the application checks and when it calls `kevent_wait()`, that call will return immediately. 

In older versions of the patch, there was no way to tell the kernel when events had been consumed out of the ring; one simply had to hope this had happened by the time the index wrapped around and events were overwritten. In the new version, instead, the application's current position is tracked, and the kernel should be occasionally informed when entries in the ring buffer are freed. That job is done with `kevent_commit()`: 

```
int kevent_commit(int ctl_fd, unsigned int new_idx, unsigned int over);
```

Here, `new_idx` is the index of the last event which has been consumed by the application. The value for `over` should be the `ring_over` field from the `kevent_ring` structure. If that value does not match what the kernel thinks it should be, the attempt to update the index will fail on the assumption that the calling process got scheduled out for a while and things happened while it was not looking. If this check were not made, confusion over index wraparound could cause events to be lost. 

As of this writing, the most significant [comment](<https://lwn.net/Articles/213681/>) is that the name "kevent" suggests an in-kernel API. The commenter (Jeff Garzik) prefers a name like "uevent" (even though there is already a subsystem which returns "uevents" in the kernel). If that remains the most substantial criticism, the kevent code might find its way into the mainline long before Evgeniy breaks the devfs record.
