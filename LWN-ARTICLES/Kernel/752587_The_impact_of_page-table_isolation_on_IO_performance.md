---
title: "The impact of page-table isolation on I/O performance"
url: https://lwn.net/Articles/752587/
date: "April 24, 2018"
category: "Security-Meltdown and Spectre"
author: "By Jonathan Corbet April 24, 2018 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 24, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Ever since [kernel page-table isolation (PTI)](<https://lwn.net/Articles/741878/>) was introduced as a mitigation for the Meltdown CPU vulnerability, users have worried about how it affects the performance of their systems. Most of that concern has been directed toward its impact on computing performance, but I/O performance also matters. At the 2018 Linux Storage, Filesystem, and Memory-Management Summit, Ming Lei presented some preliminary work he has done to try to quantify how severely PTI affects block I/O operations. 

This work was done by running the [fio benchmark](<https://github.com/axboe/fio>) on current hardware. The initial tests, running in a virtual machine, showed a significant impact: a system that could execute just over 1 million I/O operations per second (IOPS) without PTI was reduced to 726,000 IOPS with PTI turned on. The situation changes significantly when the test is run on bare metal [![\[Ming Lei\]](https://static.lwn.net/images/conf/2018/lsfmm/MingLei-sm.jpg)](<https://lwn.net/Articles/752589/>) on the same machine; in that case, a system that could achieve 1,706,000 IOPS dropped to 1,568,000 IOPS when PTI is turned on. At a little under 10%, that is a smaller impact, but still a significant one. 

It's not clear why performance regresses so severely when the test is run under virtualization. There was some theorizing that `clock_gettime()`, which is called frequently by fio, is not implemented properly on the guest system, but no real answers. 

Further tests were done using an NVMe-attached drive. In this case, the IOPS rates were about the same regardless of whether PTI was being used, but the system's CPU utilization was significantly higher in the PTI case. 

Lei concluded from his tests that enabling PTI adds about 0.2µs to the execution time of every system call. Normal synchronous I/O operations can be performed with a single system call, so they slow down slightly as a result. Asynchronous I/O operations, as used in the benchmark, require two system calls — one each to `io_submit()` and `io_getevents()`. As a result, asynchronous I/O feels the PTI penalty more severely. Interrupts add a similar penalty to each operation as well. 

Dave Hansen (who did much of the work to bring PTI to Linux) noted that there was nothing new in these results. There has always been a cost to both interrupts and system calls; PTI just makes those costs worse. He did note that it was nice to see that the IOPS don't drop when there is adequate CPU time available, though. 

Block maintainer Jens Axboe said that fio performs three `clock_gettime()` calls for every I/O operation by default. So, to a great extent, Lei's tests were measuring the impact of PTI on system-call execution time. Bart Van Assche suggested using the options that reduce the number of `clock_gettime()` calls, just as the session wound down.
