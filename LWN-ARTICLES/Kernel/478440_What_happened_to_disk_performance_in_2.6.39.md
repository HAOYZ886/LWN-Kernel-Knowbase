---
title: What happened to disk performance in 2.6.39
url: https://lwn.net/Articles/478440/
date: "January 31, 2012"
category: "Block layer-Plugging"
author: "By Jonathan Corbet January 31, 2012"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 31, 2012

Herbert Poetzl recently [reported](<https://lwn.net/Articles/478441/>) an interesting performance problem. His SSD-equipped laptop could read data at about 250MB/s with the 2.6.38 kernel, but performance dropped to 25-50MB/s on anything more recent. An order-of-magnitude performance drop is just not the sort of benefit that most people look forward to when upgrading their kernel, so this report quickly gained the attention of a number of developers. The resolution of the problem turned out to be simple, but it offers an interesting view of how high-performance disk I/O works in the kernel. 

An explanation of the problem requires just a bit of background, and, in particular, the definition of a couple of terms. "Readahead" is the process of speculatively reading file data into memory with the idea that an application is likely to want it soon. Reasonable performance when reading a file sequentially depends on proper readahead; that is the only way to ensure that reading and consuming the data can be done in parallel. Without readahead, applications will spend more time than necessary waiting for data to be read from disk. 

"Plugging," instead, is the process of stopping I/O request submissions to the low-level device for a period of time. The motivation for plugging is to allow a number of I/O requests to accumulate; that lets the I/O scheduler sort them, merge adjacent requests, and apply any sort of fairness policy that might be in effect. Without plugging, I/O requests would tend to be smaller and more scattered across the device, reducing performance even on solid-state disks. 

Now imagine that we have a process about to start reading through a long file, as indicated by your editor's unartistic rendering here: 

![\[Bad art\]](https://static.lwn.net/images/2012/plug-ra1.png)

Once the application starts reading from the beginning of the file, the kernel will set about filling the first readahead window (which is 128KB with larger files) and submit I/O for the second window, so the situation will look something like this: 

![\[Reading begins\]](https://static.lwn.net/images/2012/plug-ra2.png)

Once the application reads past 128KB into the file, the data it needs will hopefully be in memory. The readahead machinery starts up again, initiating I/O for the window starting at 256KB; that yields a situation that looks something like this: 

![\[Next window\]](https://static.lwn.net/images/2012/plug-ra3.png)

This process continues indefinitely with the kernel running to always stay ahead of the application and have the data there by the time that application gets around to reading it. 

The 2.6.39 kernel saw [some significant changes to how plugging is handled](<https://lwn.net/Articles/438256/>), with the result that the plugging and unplugging of queues is now explicitly managed in the I/O submission code. So, starting with 2.6.39, the readahead code will plug the request queue before it submits a batch of read operations, then unplug the queue at the end. The function that handles basic buffered file I/O (`generic_file_aio_read()`) also now does its own plugging. And that is where the problems begin. 

Imagine a process that is doing large (1MB) reads. As the first large read gets into `generic_file_aio_read()`, that function will plug the request queue and start working through the file pages already in memory. When it gets to the end of the first readahead window (at 128KB), the readahead code will be invoked as described above. But there's a problem: the queue is still plugged by `generic_file_aio_read()`, which is still working on that 1MB read request, so the I/O operations submitted by the readahead code are not passed on to the hardware; they just sit in the queue. 

So, when the application gets to the end of the second readahead window, we see a situation like this: 

![\[Bummer\]](https://static.lwn.net/images/2012/plug-ra4.png)

At this point, everything comes to a stop. That will cause the queue to be unplugged, allowing the readahead I/O requests to be executed at last, but it is too late. The application will have to wait. That wait is enough to hammer performance, even on solid-state devices. 

The [fix](<https://lwn.net/Articles/478459/>) is to simply remove the top-level plugging in `generic_file_aio_read()` so that readahead-originated requests can get through to the hardware. Developers who have been able to reproduce the slowdown report that this patch makes the problem go away, so this issue can be considered solved. Look for this fix to appear in a stable kernel release sometime soon.
