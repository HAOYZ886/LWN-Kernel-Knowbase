---
title: "Two performance-oriented patches: epoll and NUMA balancing"
url: https://lwn.net/Articles/913291/
date: "November 4, 2022"
category: "Epoll; Memory management-NUMA systems; NUMA"
author: "By Jonathan Corbet November 4, 2022"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 4, 2022

The search for better performance from the kernel never ends. Recently there has been a stream of smaller patches that promise incremental performance gains, at least for some types of applications. Read on for an overview of two of those patches, which make changes to the epoll system calls and to NUMA balancing. This work shows where developers are looking for performance improvements — and that not everybody measures performance the same way. 

#### An epoll optimization

The [epoll](<https://man7.org/linux/man-pages/man7/epoll.7.html>) family of system calls is aimed at event-loop applications that manage large numbers of file descriptors. Unlike [`poll()`](<https://man7.org/linux/man-pages/man2/poll.2.html>) or [`select()`](<https://man7.org/linux/man-pages/man2/select.2.html>), epoll allows the per-file-descriptor setup be performed once (with [`epoll_ctl()`](<https://man7.org/linux/man-pages/man2/epoll_ctl.2.html>)); the result can then be used with multiple [`epoll_wait()`](<https://man7.org/linux/man-pages/man2/epoll_wait.2.html>) calls to poll for new events. That reduces the overall polling overhead significantly, especially as the number of file descriptors being watched grows. The epoll system calls add some complexity, but for applications where per-event performance matters, it is worth the trouble. 

Normally, `epoll_wait()` will block the calling process until at least one of the polled file descriptors is ready for I/O. There is a timeout parameter, though, that can be used to limit the time the application will remain blocked. What is lacking, however, is a way to specify a _minimum_ time before the `epoll_wait()` call returns. That may not be surprising; as a general rule, nobody wants to increase an application's latency unnecessarily, so `epoll_wait()` is designed to return quickly when there is something to be done. 

Even so, [this patch set from Jens Axboe](<https://lwn.net/ml/linux-kernel/20221030220203.31210-1-axboe@kernel.dk/>) adds this minimum wait time. It creates a new `epoll_ctl()` operation, `EPOLL_CTL_MIN_WAIT`, to specify the shortest time that subsequent `epoll_wait()` calls should block before returning to user space. The reasoning behind this seemingly counterintuitive capability is to increase the number of events that can be returned by each `epoll_wait()` call. Even with much of the setup work taken out, each system call still has a cost. In situations where numerous events can be expected to arrive within a short time period, it can make sense to wait for a few of them to show up and only pay the system-call cost once. 

In other words, an application may want to trade off a bit of latency for better throughput overall. This is seemingly a common use case; as Axboe [put it](<https://lwn.net/ml/linux-kernel/20221030220203.31210-7-axboe@kernel.dk/>): 

> For medium workload efficiencies, some production workloads inject artificial timers or sleeps before calling epoll_wait() to get better batching and higher efficiencies. While this does help, it's not as efficient as it could be. 

Using this feature, he said, can reduce CPU usage by 6-7%. Axboe is seeking input on the API; specifically, whether the minimum timeout should be set once with `epoll_ctl()`, or whether it should instead be provided with each `epoll_wait()` call. This would be a good time for potential users to make their preferences known. 

#### Control over NUMA balancing

Non-uniform memory access (NUMA) systems are characterized by variable RAM access times; memory that is attached to the node on which a given thread is running will be faster for that thread to access than memory on other nodes in the system. On such systems, applications will thus perform better if their memory is located on the nodes they are running on. To make that happen, the kernel performs NUMA balancing — moving pages within the system so that they are resident on the nodes where they are actually being used. 

NUMA balancing, when done correctly, improves the throughput of the system by increasing memory speeds. But NUMA balancing can also cause short-term latency spikes for applications, especially if they incur page faults while the kernel is migrating pages across nodes. The culprit here, [as is often the case](<https://lwn.net/Articles/893906/>), is contention for the process's `mmap_lock`. For some latency-sensitive applications, that can be a problem; there are, seemingly, applications where it is better to pay the cost of suboptimal memory placement to avoid being stalled during NUMA balancing. 

For such applications, Gang Li has posted [a patch set](<https://lwn.net/ml/linux-kernel/20221027025302.45766-1-ligang.bdlg@bytedance.com/>) adding a new [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) operation, `PR_NUMA_BALANCING`, that can control whether NUMA balancing is performed for the calling process. If that process disables NUMA balancing, pages will be left where they are even at the cost of longer access times. Benchmark results included in the cover letter show that the performance effects of disabling NUMA balancing vary considerably depending on the workload. This feature will not be useful for many applications, but there are seemingly some that will benefit. 

The kernel development community tries hard to minimize the number of tuning knobs that it adds to the kernel. Each of those knobs is a maintenance burden for the community but, more importantly, tuning knobs are a burden for application developers and system administrators as well. It can be difficult for those users to even discover all of the parameters that are available, much less set them for optimal performance. It is better for the kernel to tune itself for the best results whenever possible. 

Patches like the above show that this self-tuning is not always possible, at least in the current state of the art. Achieving the best performance for all applications gets harder when different applications need to optimize different metrics. Thus, one of these patches allows developers to prioritize throughput over latency, while the other does the opposite. This diversity of requirements seemingly ensures that anybody wanting to get that last bit of performance out of their application will continue to need to play with tuning knobs.
