---
title: Toward a kernel events interface
url: https://lwn.net/Articles/193691/
date: "August 1, 2006"
category: Events reporting; Kevent
author: 
---

[Last week's article](<http://lwn.net/Articles/192767/>) on network channels suggested that channels might not be the way of the future at all. Since then, there has been a great deal of discussion on how networking might move forward on many levels, some of which might yet include channels. Your editor plans to gain an understanding of the Grand Unified Flow Cache and related concepts (such as Rusty's plans to thrash up netfilter yet again) for a future article; for now, we'll look at a different aspect of networking (and beyond): a user-space events interface. 

Unlike some other operating systems, Linux currently lacks a system call for generalized event reporting. Linux applications, instead, use calls like `poll()` to figure out when there is work to be done. Unfortunately, `poll()` does not solve the entire problem, so application event loops must do complicated things to deal with things like signals. Handling asynchronous I/O within a traditional Linux event loop can be especially tricky. If there were a single interface which provided an application with all of the event information it needed, applications would get simpler. There is also the potential for significant performance improvements. 

There are two active proposals for event interfaces for Linux: the [kevent mechanism](<http://lwn.net/Articles/172844/>) and the event channel API [proposed by Ulrich Drepper](<http://lwn.net/Articles/192410/>) at this year's Ottawa Linux Symposium. Of the two, kevents currently have the advantage for one simple reason: there is an existing, working implementation to look at. So most of the discussion has concerned how kevents can be improved. 

The original kevent API is seen as being a bit difficult; it relies on a single multiplexer system call (`kevent_ctl()`), an approach which is generally frowned upon. The call also requires the application to construct an array with two different types of structures, which is a bit awkward. So one of the first suggestions has been to separate out various parts of the API. The [current kevent patch](<http://lwn.net/Articles/193656/>) (as of August 1) contains a new system call: 

```
int kevent_get_events(int ctl_fd, 
                              unsigned int min_nr,
    			  unsigned int max_nr,
    			  unsigned int timeout,
    			  void *buf,
    			  unsigned flags);
```

This call would return between `min_nr` and `max_nr` events, storing them sequentially in `buf`, subject to the given `timeout` (specified in milliseconds). The `flags` argument is unused in the current implementation. 

There are a number of things which might be improved with this interface, but, as it happens, its final form is likely to look quite different. The current interface still requires frequent system calls to retrieve events; Linux system calls are fast, but, in a high-bandwidth situation, it still would be preferable to spend more time in user space if possible. With a different approach to event reporting, it might just be possible. 

The idea which has been discussed is to map an array of `kevent` structures between kernel and user space. This array would be treated as a circular buffer, perhaps managed using a cache-friendly, channel-like index mechanism. The kernel would place events into the buffer when they occur, and user-space would consume them. Whenever there are events to process, the application could obtain them without entering the kernel at all. Once this mechanism is in place, the `kevent_get_events()` call could go away, replaced by a simple "wait for events" interface (though glibc would almost certainly provide a synchronous "get events" function). The result should be a very fast interface, especially when the number of events is large. 

There are a couple of issues to be worked out, still. One has to do with what happens when the buffer fills. The current asynchronous I/O interface does not allow there to be more outstanding operations than there are available control block structures; that way, there is guaranteed to be space to report on the status of each operation. That can be important, since the place in the kernel which wants to do the reporting is often running at software or hardware interrupt level. If one envisions using kevents to track thousands of open sockets, an unknown number of connection events, etc., however, preallocating all of the event structures becomes increasingly impractical. So something intelligent will have to be done when the buffer fills. 

The other issue has to do with "level-triggered" events which correspond more to a specific status than a real event which has occurred. "This socket can be written to" is such an event. When an interface like `poll()` is used to query whether a write would block, the kernel can check the status and return immediately if the given file descriptor can be written to. Reporting this sort of status through a circular buffer is rather harder to do. So, one way or another, applications will have to explicitly poll for such events. 

Given the current level of interest, some way of dealing with these issues seems likely to surface in the near future. That could clear the path for merging kevents into the mainline, perhaps as early as 2.6.20.
