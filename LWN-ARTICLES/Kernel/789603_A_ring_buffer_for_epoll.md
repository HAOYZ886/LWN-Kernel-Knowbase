---
title: A ring buffer for epoll
url: https://lwn.net/Articles/789603/
date: "May 30, 2019"
category: Epoll
author: "By Jonathan Corbet May 30, 2019"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 30, 2019

The set of system calls known collectively as [epoll](<http://man7.org/linux/man-pages/man7/epoll.7.html>) was designed to make polling for I/O events more scalable. To that end, it minimizes the amount of setup that must be done for each system call and returns multiple events so that the number of calls can also be minimized. But that turns out to still not be scalable enough for some users. The response to this problem, in the form of [this patch series](<https://lwn.net/ml/linux-kernel/20190516085810.31077-1-rpenyaev@suse.de/>) from Roman Penyaev, takes a familiar form: add yet another ring-buffer interface to the kernel. 

The `poll()` and `select()` system calls can be used to wait until at least one of a set of file descriptors is ready for I/O. Each call, though, requires the kernel to set up an internal data structure so that it can be notified when any given descriptor changes state. Epoll gets around this by separating the setup and waiting phases, and keeping the internal data structure around for as long as it is needed. 

An application starts by calling [`epoll_create1()`](<http://man7.org/linux/man-pages/man2/epoll_create1.2.html>) to create a file descriptor to use with the subsequent steps. That call, incidentally, supersedes `epoll_create()`; it replaces an unused argument with a flags parameter. Then `[epoll_ctl()](<http://man7.org/linux/man-pages/man2/epoll_ctl.2.html>)` is used to add individual file descriptors to the set monitored by epoll. Finally, a call to `[epoll_wait()](<http://man7.org/linux/man-pages/man2/epoll_wait.2.html>)` will block until at least one of the file descriptors of interest has something to report. This interface is a bit more work to use than `poll()`, but it makes a big difference for applications that are monitoring huge numbers of file descriptors. 

That said, it would seem that there is still room for doing things better. Even though epoll is more efficient than its predecessors, an application still has to make a system call to get the next set of file descriptors that are ready for I/O. On a busy system, where there is almost always something that is needing attention, it would be more efficient if there were a way to get new events without calling into the kernel. That is where Penyaev's patch set comes in; it creates a ring buffer shared between the application and the kernel that can be used to transmit events as they happen. 

#### epoll_create() — the third time is the charm

The first step for an application that wishes to use this mechanism is to tell the kernel that polling will be used and how big the ring buffer should be. Of course, `epoll_create1()` does not have a parameter that can be used for the size information, so it is necessary to add `epoll_create2()`: 

```
int epoll_create2(int flags, size_t size);
```

There is a new flag, `EPOLL_USERPOLL`, that tells the kernel to use a ring buffer to communicate events; the `size` parameter says how many entries the ring buffer should hold. This size will be rounded up to the next power of two; the result sets an upper bound on the number of file descriptors that this epoll instance will be able to monitor. A maximum of 65,536 entries is enforced by the current patch set. 

File descriptors are then added to the polling set in the usual way with `epoll_ctl()`. There are some restrictions that apply here, though, since some modes of operation are not compatible with user-space polling. In particular, every file descriptor must request edge-triggered behavior with the `EPOLLET` flag. Only one event will be added to the ring buffer when a file descriptor signals readiness; continually adding events for level-triggered behavior clearly would not work well. The `EPOLLWAKEUP` flag (which can be used to prevent system suspend while specific events are being processed) does not work in this mode; `EPOLLEXCLUSIVE` is also not supported. 

Two or three separate `mmap()` calls are required to map the ring buffer into user space. The first one should have an offset of zero and a length of one page; it will yield a page containing this structure: 

```
struct epoll_uheader {
    	u32 magic;          /* epoll user header magic */
    	u32 header_length;  /* length of the header + items */
    	u32 index_length;   /* length of the index ring, always pow2 */
    	u32 max_items_nr;   /* max number of items */
    	u32 head;           /* updated by userland */
    	u32 tail;           /* updated by kernel */
    
    	struct epoll_uitem items[];
        };
```

The `header_length` field, somewhat confusingly, contains the length of both the `epoll_uheader` structure and the `items` array. As seen in [this example program](<https://github.com/rouming/test-tools/blob/master/userpolled-epoll.c>), the intended use pattern appears to be that the application will map the header structure, get the real length, unmap the just-mapped page, then remap it using `header_length` to get the full `items` array. 

One might expect that `items` is the ring buffer, but there is a layer of indirection used here. Getting at the actual ring buffer requires calling `mmap()` another time with `header_length` as the offset and the `index_length` header field as the length. The result will be an array of integer indexes into the `items` array that functions as the real ring buffer. 

The actual items used to indicate events are represented by this structure: 

```
struct epoll_uitem {
    	__poll_t ready_events;
    	__poll_t events;
    	__u64 data;
        };
```

Here, `events` appears to be the set of events that was requested when `epoll_ctl()` was called, and `ready_events` is the set of events that has actually happened. The `data` field comes through directly from the `epoll_ctl()` call that added this file descriptor. 

Whenever the `head` and `tail` fields differ, there is at least one event to be consumed from the ring buffer. To consume an event, the application should read the entry from the index array at `head`; this read should be performed in a loop until a non-zero value is found there. The loop, evidently, is required to wait, if necessary, until the kernel's write to that entry is visible. The value read is an index into the `items` array — almost. It is actually the index plus one. The data should be copied from the entry and `ready_events` set to zero; then the `head` index should be incremented. 

So, in a cleaned up form, code that reads from the ring buffer will look something like this: 

```
while (header->tail == header->head)
            ;  /* Wait for an event to appear */
        while (index[header->head] == 0)
            ;  /* Wait for event to really appear */
        item = header->items + index[header->head] - 1;
        data = item->data;
        item->ready_events = 0;  /* Mark event consumed */
        header->head++;
```

In practice, this code is likely to be using C atomic operations rather than direct reads and writes, and `head` must be incremented in a circular fashion. But hopefully the idea is clear. 

Busy-waiting on an empty ring buffer is obviously not ideal. Should the application find itself with nothing to do, it can still call `epoll_wait()` to block until something happens. This call will only succeed, though, if the `events` array is passed as `NULL`, and `maxevents` is set to zero; in other words, `epoll_wait()` will block, but it will not, itself, return any events to the caller. It will, though, helpfully return `ESTALE` to indicate that there are events available in the ring buffer. 

This patch set is in its third revision, and there appears to be little opposition to its inclusion at this point. The work has not yet found its way into linux-next, but it still seems plausible that it could be deemed ready for the 5.3 merge window. 

#### Some closing grumbles

Figuring out the above interface required a substantial amount of reverse engineering of the code. This is a rather complex new API, but it is almost entirely undocumented; that will make it hard to use, but the lack of documentation also makes it hard to review the API in the first place. It is doubtful that anybody beyond the author has written any code to use this API at this point. Whether the development community will fully understand this API before committing to it is far from clear. 

Perhaps the saddest thing, though, is that this will be yet another of many ring-buffer interfaces in the kernel. Others include perf events, ftrace, io_uring, `AF_XDP` and, doubtless, others that don't come immediately to mind. Each of these interfaces has been created from scratch and must be understood (and consumers implemented) separately by user-space developers. Wouldn't it have been nice if the kernel had defined a set of standards for ring buffers shared with user space rather than creating something new every time? One cannot blame the current patch set for this failing; that ship sailed some time ago. But it does illustrate a shortcoming in how Linux kernel APIs are designed; they seem doomed to never fit into a coherent and consistent whole.
