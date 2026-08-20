---
title: Multiple kernels on a single system
url: https://lwn.net/Articles/1038847/
date: "September 19, 2025"
category: "Multi-kernel"
author: "By Jonathan Corbet September 19, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 19, 2025

The Linux kernel generally wants to be in charge of the system as a whole; it runs on all of the available CPUs and controls access to them globally. Cong Wang has just come forward with [a different approach](<https://lwn.net/ml/all/20250918222607.186488-1-xiyou.wangcong@gmail.com>): allowing each CPU to run its own kernel. The patch set is in an early form, but it gives a hint for what might be possible. 

The patch set as a whole only touches 1,400 lines of code, adding a few basic features; there would clearly need to be a lot more work done to make this feature useful. The first part is a new `KEXEC_MULTIKERNEL` flag to the [`kexec_load()`](<https://man7.org/linux/man-pages/man2/kexec_load.2.html>) system call, requesting that a new kernel be booted on a specific CPU. That CPU must be in the offline state when the call is made, or the call will fail with an `EBUSY` error. It would appear that it is only possible to assign a single CPU to any given kernel in this mode; the current interface lacks a way to specify more than one CPU. There is a bunch of x86-64 assembly magic to set up the target CPU for the new kernel and to boot it there. 

The other significant piece is a new inter-kernel communication mechanism, based on inter-processor interrupts, that allows the kernels running on different CPUs to talk to each other. Shared memory areas are set aside for the efficient movement of data between the kernels. While the infrastructure is present in the patch set, there are no users of it in this series. A real-world system running in this mode would clearly need to use this communication infrastructure to implement a lot of coordination of resources to keep the kernels from stepping on each other, but that work has not been posted yet. 

The final patches in the series add a new `/proc/multikernel` file that can be used to monitor the state of the various kernels running in the system. 

Why would one want to do this? In the cover letter, Wang mentions a few advantages, including improved fault isolation and security, better efficiency than virtualization, and the ease of zero-downtime updates in conjunction with the [kexec handover](<https://lwn.net/Articles/1033364/>) mechanism. He also mentions the ability to run special-purpose kernels (such as a realtime kernel) for specific workloads. 

The work that has been posted is clearly just the beginning: 

> This patch series represents only the foundational framework for multikernel support. It establishes the basic infrastructure and communication mechanisms. We welcome the community to build upon this foundation and develop their own solutions based on this framework. 

The new files in the series carry copyright notices for [Multikernel Technologies Inc](<https://multikernel.io/>), which, seemingly, is also developing its own solutions based on this code. In other words, this looks like more than a hobby project; it will be interesting to see where it goes from here. Perhaps this relatively old idea (Larry McVoy was [proposing](<https://lwn.net/Articles/4536/>) "cache-coherent clusters" for Linux at least as far back as 2002) will finally come to fruition.
