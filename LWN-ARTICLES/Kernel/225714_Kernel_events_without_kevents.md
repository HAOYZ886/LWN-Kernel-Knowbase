---
title: Kernel events without kevents
url: https://lwn.net/Articles/225714/
date: "March 13, 2007"
category: Kevent; eventfs; signalfd; timerfd
author: 
---

The long story of the kevent subsystem has appeared on this page a number of times. Kevents are designed to give applications a single system call which they can use to wait for any events of interest: I/O, timers, signals, and more. While quite a bit of work has been done on this code, its path into the kernel has been long. A number of developers are still unconvinced that the interface is needed, and, if it is, that the proposed kevent API (which would have to be maintained forever) is the right one. Now there is a competing approach which may prove easier for the community to accept. 

Davide Libenzi is the creator of the `epoll_wait()` system call; it is a version of `poll()` which is intended to be scalable to large numbers of file descriptors. This API seems to be well regarded for what it does, but it is limited to waiting on file descriptors. Many of the things that kevents address are not associated with files, and so cannot be handled through the epoll interface. 

Kevents fix that shortcoming with the creation of a new subsystem and user-space API. Davide has now shown up with a different strategy: make a way for applications to request delivery of events via a file descriptor. Consider, for example, the case of signals. Signals tend to be tricky for applications to handle; they are asynchronous events which are delivered to a special signal handler function, but that function is seriously limited in what it can do. In response, application developers have resorted to tricks like writing a byte to an internal pipe so that the signal can be handled in the main event loop. 

Davide has proposed [a new system call](<http://lwn.net/Articles/225648/>) named `signalfd()` which can help developers avoid much of the hassle of working with signals: 

```
int signalfd(int ufd, const sigset_t *mask, size_t masksize);
```

If `ufd` is `-1`, this call will create (and return) a new file descriptor. The signals described in `mask` will be caught and returned to the process by way of that file descriptor. It is pollable, allowing signals to be handled in an event loop based on `select()`, `poll()` or `epoll_wait()`. When signals are available, they can be read from the descriptor as data; the `signalfd_siginfo` structure returned by `read()` has the signal number and all of the related information that comes with it. 

If `ufd` is set to an existing signal file descriptor, the `signalfd()` call will change to the new `mask`. It is worth noting that reading from this file descriptor competes with normal signal delivery for queued signals; there is no way to predict whether the signal will be delivered in the usual way or will be read from the file descriptor. This situation can be avoided by using `sigprocmask()` to block normal delivery of the signal(s) of interest. 

There is [a similar interface for timer events](<http://lwn.net/Articles/225646/>): 

```
int timerfd(int ufd, int clockid, int timertype, 
                    const struct timespec *when);
```

Once again, `ufd` is `-1` to create a new file descriptor, or an existing timer file descriptor which is to be modified. The `clockid` parameter describes which clock is wanted: `CLOCK_MONOTONIC` or `CLOCK_REALTIME`. The type of timer is described by `timertype`: `TFD_TIMER_REL` for a time relative to the current time, `TFD_TIMER_ABS` for an absolute time, or `TFD_TIMER_SEQ` for a repeating timer at a given interval. The `when` structure contains the requested expiration time. 

Once again, this file descriptor can be polled. Reading from it yields an integer value saying how many times the timer has fired since the last time it was read. 

Evgeniy Polyakov, the author of the kevent patches, has not been sitting still while these patches have gone around. His proposal is called [eventfs](<http://lwn.net/Articles/225592/>); it is a special filesystem which offers the ability to bind events to file descriptors. The first version of the patch only handles signals, via a system call named (yes) `signalfd()`: 

```
int signalfd(int signal, int flags);
```

This call creates a new file descriptor for the given `signal` (a separate file descriptor is required for each signal in this scheme). In the current code, if `flags` is nonzero, the signal will only be delivered through eventfs and will never go into the signal queue. The file descriptor is pollable, but there is no way to read any information from it. So any associated signal information is lost; multiple deliveries of the same signal between polls will also be lost. 

One assumes that Evgeniy's patches could be improved over time, but Davide's version seems to be ahead in terms of features, coverage, and community review. Davide has also avoided the need to create a new filesystem to back the whole thing up. So if bets were being taken on which approach might make it into the kernel, Davide would seem to be in the lead at the moment. 

There are certainly things to be said for this approach. It brings Linux toward a single interface for event delivery without the need for a new, complex API. It also reinforces the role of the file descriptor as a fundamental object for interaction with the kernel. On the other hand, the poll interfaces do not provide a way for applications to receive events without the need to call into the kernel - a feature which has been requested by some interested parties. There are also event types (asynchronous I/O completion, for example) which are not yet covered. So, if things do go this way, it would not be surprising to see patches trying to fill in those gaps in the near future.
