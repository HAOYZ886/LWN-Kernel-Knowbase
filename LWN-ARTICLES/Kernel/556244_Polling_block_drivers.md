---
title: Polling block drivers
url: https://lwn.net/Articles/556244/
date: "June 26, 2013"
category: Block layer
author: "By Jonathan Corbet June 26, 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 26, 2013

The number of latency-sensitive applications running on Linux seems to be increasing, with the result that more latency-related changes are finding their way into the kernel. Recently LWN looked at the [Ethernet device polling patch set](<https://lwn.net/Articles/551284/>), which implements polling to provide minimal latency to critical networking tasks. But what happens if you want the lowest possible latency for block I/O requests instead? Matthew Wilcox's [block driver polling patch](<https://lwn.net/Articles/555886/>) is an attempt to answer that question. 

As Matthew [put it](<https://lwn.net/Articles/556245/>), there are users who are willing to go to great lengths to lower the latencies they experience with block I/O requests: 

The problem is that some of the people who are looking at those technologies are crazy. They want to "bypass the kernel" and "do user space I/O" because "the kernel is too slow". This patch is part of an effort to show them how crazy they are. 

The patch works by adding a new driver callback to `struct backing_dev_info`: 

```
int (*io_poll)(struct backing_dev_info *bdi);
```

This function, if present, should poll the given device for completed I/O operations. If any are found, they should be signaled to the block layer; the return value is the number of operations found (or a negative error code). 

Within the block layer, the `io_poll()` function will be called whenever a process is about to sleep waiting for an outstanding operation. By placing the poll calls there, Matthew hopes to avoid going into polling when there is other work to be done; it allows, for example, the submission of multiple operations without invoking the poll loop. But, once a process actually needs the result of a submitted operation, it begins polling rather than sleep. 

Polling continues until one of a number of conditions comes about. One of those, of course, is that an operation that the current process is waiting for completes. In the absence of a completed operation, the process will continue polling until it receives a signal or the scheduler indicates that it would like to switch to a different process. So, in other words, polling will stop if a higher-priority process becomes runnable or if the current process exhausts its time slice. Thus, while the polling happens in the kernel, it is limited by the relevant process's available CPU time. 

Linus [didn't like this approach](<https://lwn.net/Articles/556248/>), saying that the polling still wastes CPU time even if there is no higher-priority process currently contending for the CPU. That said, he's not necessarily opposed to polling; he just does not want it to happen if there might be other runnable processes. So, he suggested, the polling should be moved to the idle thread. Then polling would only happen when the CPU was about to go completely idle, guaranteeing that it would not get in the way of any other process that had work to do. 

But Linus might actually lose in this case. Block maintainer Jens Axboe [responded](<https://lwn.net/Articles/556250/>) that an idle-thread solution would not work. ""If you need to take the context switch, then you've negated pretty much all of the gain of the polled approach."" Also he [noted](<https://lwn.net/Articles/556251/>) that the current patch does the polling in (almost) the right place, just where the necessary information is available. So Jens appears to be disposed toward merging something that looks like the current patch; at that point, Linus will likely accept it. 

But Jens did ask for a bit more smarts when it comes to deciding when the polling should be done; in the current patch, it happens unconditionally for any device that provides an `io_poll()` function. A better approach, he said, would be to provide a way for specific processes to opt in to the polling, since, even on latency-sensitive systems, polling will not be needed by all processes. Those processes that do not need extremely low latency should not have to give up some of their allotted CPU time for I/O polling. 

So the patch will certainly see some work before it is ready for merging. But the benefits are real: in a test run by Matthew on an [NVMe](<http://www.nvmexpress.org/>) device, I/O latencies dropped from about 8.0µs to about 5.5µs — a significant reduction. The benefit will only become more pronounced as the speed of solid-state storage devices increases; as the time required for an I/O operation approaches 1µs, an extra 2.5µs of overhead will come to dominate the picture. Latency-sensitive users will seek to eliminate that overhead somehow; addressing it in the kernel is a good way to ensure that all users are able to take advantage of this work.
